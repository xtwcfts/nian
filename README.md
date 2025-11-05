import os
import random
import string
import qrcode
from io import BytesIO
from flask import Flask, render_template_string, request, redirect, url_for, session, jsonify, send_file, flash
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime, timedelta
import pymysql

app = Flask(__name__)
app.config['SECRET_KEY'] = 'sudoku_secret_key_2025'
app.config['SQLALCHEMY_DATABASE_URI'] = 'mysql+pymysql://root:123456@localhost:3308/sudoku_db?charset=utf8mb4'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)

# -------------------------- 数据库模型 --------------------------
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), unique=True, nullable=False)
    password = db.Column(db.String(50), nullable=False)  # 简化处理，实际项目需加密
    is_admin = db.Column(db.Boolean, default=False)
    is_active = db.Column(db.Boolean, default=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)

class SudokuLevel(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    level_name = db.Column(db.String(100), nullable=False)
    difficulty = db.Column(db.String(20), nullable=False)  # easy/medium/hard
    puzzle = db.Column(db.Text, nullable=False)  # 字符串存储，0表示空格
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    created_by = db.Column(db.Integer, db.ForeignKey('user.id'))

class SolvedRecord(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    level_id = db.Column(db.Integer, db.ForeignKey('sudoku_level.id'), nullable=False)
    time_spent = db.Column(db.Integer, nullable=False)  # 耗时（秒）
    solved_at = db.Column(db.DateTime, default=datetime.utcnow)
    
    # 关联关系（便于查询）
    user = db.relationship('User', backref=db.backref('records', lazy=True))
    sudoku_level = db.relationship('SudokuLevel', backref=db.backref('records', lazy=True))

class QRLogin(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    token = db.Column(db.String(100), unique=True, nullable=False)
    # 关键修改：允许 user_id 为空，等待确认时填充
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=True)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    used = db.Column(db.Boolean, default=False)

# -------------------------- 数独核心函数 --------------------------
def generate_sudoku(difficulty='medium'):
    """生成指定难度的数独题目"""
    base = [1,2,3,4,5,6,7,8,9]
    random.shuffle(base)
    # 基础数独网格（保证合法）
    grid = [
        [base[0], base[1], base[2], base[3], base[4], base[5], base[6], base[7], base[8]],
        [base[3], base[4], base[5], base[6], base[7], base[8], base[0], base[1], base[2]],
        [base[6], base[7], base[8], base[0], base[1], base[2], base[3], base[4], base[5]],
        [base[1], base[2], base[0], base[4], base[5], base[3], base[7], base[8], base[6]],
        [base[4], base[5], base[3], base[7], base[8], base[6], base[1], base[2], base[0]],
        [base[7], base[8], base[6], base[1], base[2], base[0], base[4], base[5], base[3]],
        [base[2], base[0], base[1], base[5], base[3], base[4], base[8], base[6], base[7]],
        [base[5], base[3], base[4], base[8], base[6], base[7], base[2], base[0], base[1]],
        [base[8], base[6], base[7], base[2], base[0], base[1], base[5], base[3], base[4]]
    ]
    # 按难度移除数字（难度越高移除越多）
    remove_count = {'easy':30, 'medium':40, 'hard':50}[difficulty]
    for _ in range(remove_count):
        row = random.randint(0, 8)
        col = random.randint(0, 8)
        while grid[row][col] == 0:  # 避免重复移除
            row = random.randint(0, 8)
            col = random.randint(0, 8)
        grid[row][col] = 0
    return ''.join(str(cell) for row in grid for cell in row)

def validate_sudoku(puzzle, solution):
    """验证数独答案是否正确"""
    try:
        # 1. 检查长度（必须都是81位）
        if len(puzzle) != 81 or len(solution) != 81:
            print(f"验证失败：长度错误（puzzle: {len(puzzle)}, solution: {len(solution)}）")
            return False
        
        # 2. 检查是否有未填项
        if '0' in solution:
            print("验证失败：答案包含未填项（0）")
            return False
        
        # 3. 转换为二维网格
        puzzle_grid = [list(puzzle[i*9:(i+1)*9]) for i in range(9)]
        solution_grid = [list(solution[i*9:(i+1)*9]) for i in range(9)]
        
        # 4. 检查原始数字是否被修改
        for i in range(9):
            for j in range(9):
                if puzzle_grid[i][j] != '0' and puzzle_grid[i][j] != solution_grid[i][j]:
                    print(f"验证失败：原始数字被修改（位置({i},{j})，原:{puzzle_grid[i][j]}, 新:{solution_grid[i][j]}）")
                    return False
        
        # 5. 检查行、列、3x3宫格唯一性
        def is_unique(arr):
            return len(set(arr)) == 9 and all(c in '123456789' for c in arr)
        
        # 检查行
        for i in range(9):
            if not is_unique(solution_grid[i]):
                print(f"验证失败：行{i}存在重复或无效数字")
                return False
        
        # 检查列
        for j in range(9):
            column = [solution_grid[i][j] for i in range(9)]
            if not is_unique(column):
                print(f"验证失败：列{j}存在重复或无效数字")
                return False
        
        # 检查3x3宫格
        for i in range(0, 9, 3):
            for j in range(0, 9, 3):
                box = []
                for x in range(3):
                    for y in range(3):
                        box.append(solution_grid[i+x][j+y])
                if not is_unique(box):
                    print(f"验证失败：宫格({i//3},{j//3})存在重复或无效数字")
                    return False
        
        return True
    except Exception as e:
        print(f"验证函数错误：{str(e)}")
        return False

# -------------------------- 二维码相关函数 --------------------------
def generate_token():
    """生成随机二维码token（32位字母+数字）"""
    return ''.join(random.choices(string.ascii_letters + string.digits, k=32))

def generate_qr_code(token, request):
    """根据token生成登录二维码（返回BytesIO对象）"""
    login_url = f"{request.host_url}qr_login_confirm/{token}"
    qr = qrcode.QRCode(version=1, box_size=10, border=5)
    qr.add_data(login_url)
    qr.make(fit=True)
    img = qr.make_image(fill='black', back_color='white')
    # 保存到内存缓冲区
    buffer = BytesIO()
    img.save(buffer, format='PNG')
    buffer.seek(0)
    return buffer

# -------------------------- 数据库初始化 --------------------------
def init_database():
    """自动创建数据库、表、管理员账号和初始关卡"""
    try:
        # 1. 先连接MySQL服务器，创建数据库（避免数据库不存在的错误）
        conn = pymysql.connect(
            host='localhost',
            user='root',
            password='123456',
            port=3308,
            charset='utf8mb4'
        )
        cursor = conn.cursor()
        cursor.execute("CREATE DATABASE IF NOT EXISTS sudoku_db")  # 自动创建数据库
        cursor.execute("USE sudoku_db")
        conn.close()
        
        # 2. 创建所有表结构
        db.create_all()
        
        # 3. 创建管理员账号（admin/admin，仅首次创建）
        if not User.query.filter_by(username='admin').first():
            admin = User(username='admin', password='admin', is_admin=True)
            db.session.add(admin)
            db.session.commit()
        
        # 4. 创建初始关卡（3个难度，仅首次创建）
        if not SudokuLevel.query.first():
            init_levels = [
                ('简单关卡 - 入门', 'easy'),
                ('中等关卡 - 进阶', 'medium'),
                ('困难关卡 - 挑战', 'hard')
            ]
            for name, diff in init_levels:
                puzzle = generate_sudoku(diff)
                level = SudokuLevel(
                    level_name=name,
                    difficulty=diff,
                    puzzle=puzzle,
                    created_by=1  # 管理员ID（固定为1，因为首次创建admin时ID=1）
                )
                db.session.add(level)
            db.session.commit()
        
        print("✅ 数据库初始化成功（含数据库、表、管理员账号、初始关卡）")
    except Exception as e:
        print(f"❌ 数据库初始化错误: {str(e)}")

# -------------------------- 模板定义（修复所有错误） --------------------------
templates = {
    # 登录页面
    'login.html': '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>数独游戏 - 登录</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <style>
            * { margin: 0; padding: 0; box-sizing: border-box; }
            body { font-family: "Arial", sans-serif; max-width: 450px; margin: 50px auto; padding: 0 20px; }
            h2 { text-align: center; margin-bottom: 30px; color: #333; }
            .form-group { margin-bottom: 20px; }
            label { display: block; margin-bottom: 8px; color: #666; font-weight: 500; }
            input { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 4px; font-size: 16px; }
            button { width: 100%; padding: 12px; background: #4CAF50; color: white; border: none; border-radius: 4px; font-size: 16px; cursor: pointer; }
            button:hover { background: #45a049; }
            .links { margin-top: 20px; text-align: center; }
            a { color: #4CAF50; text-decoration: none; margin: 0 8px; }
            .flash { color: #f44336; text-align: center; margin-bottom: 20px; padding: 10px; border: 1px solid #fadbd8; border-radius: 4px; }
        </style>
    </head>
    <body>
        <h2>数独游戏 - 登录</h2>
        {% with messages = get_flashed_messages() %}
          {% if messages %}
            {% for msg in messages %}
              <div class="flash">{{ msg }}</div>
            {% endfor %}
          {% endif %}
        {% endwith %}
        <form method="post">
            <div class="form-group">
                <label>用户名</label>
                <input type="text" name="username" required placeholder="请输入用户名">
            </div>
            <div class="form-group">
                <label>密码</label>
                <input type="password" name="password" required placeholder="请输入密码">
            </div>
            <button type="submit">登录</button>
        </form>
        <div class="links">
            <a href="/register">注册账号</a> | 
            <a href="/forgot_password">忘记密码</a> |
            <a href="/qr_login">二维码登录</a>
    </body>
    </html>
    ''',

    # 注册页面
    'register.html': '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>数独游戏 - 注册</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <style>
            * { margin: 0; padding: 0; box-sizing: border-box; }
            body { font-family: "Arial", sans-serif; max-width: 450px; margin: 50px auto; padding: 0 20px; }
            h2 { text-align: center; margin-bottom: 30px; color: #333; }
            .form-group { margin-bottom: 20px; }
            label { display: block; margin-bottom: 8px; color: #666; font-weight: 500; }
            input { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 4px; font-size: 16px; }
            button { width: 100%; padding: 12px; background: #4CAF50; color: white; border: none; border-radius: 4px; font-size: 16px; cursor: pointer; }
            button:hover { background: #45a049; }
            .links { margin-top: 20px; text-align: center; }
            a { color: #4CAF50; text-decoration: none; }
            .flash { color: #f44336; text-align: center; margin-bottom: 20px; padding: 10px; border: 1px solid #fadbd8; border-radius: 4px; }
        </style>
    </head>
    <body>
        <h2>数独游戏 - 注册</h2>
        {% with messages = get_flashed_messages() %}
          {% if messages %}
            {% for msg in messages %}
              <div class="flash">{{ msg }}</div>
            {% endfor %}
          {% endif %}
        {% endwith %}
        <form method="post">
            <div class="form-group">
                <label>用户名</label>
                <input type="text" name="username" required placeholder="请输入用户名">
            </div>
            <div class="form-group">
                <label>密码</label>
                <input type="password" name="password" required placeholder="请输入密码">
            </div>
            <div class="form-group">
                <label>确认密码</label>
                <input type="password" name="confirm_password" required placeholder="请再次输入密码">
            </div>
            <button type="submit">注册</button>
        </form>
        <div class="links">
            <a href="/login">已有账号？返回登录</a>
        </div>
    </body>
    </html>
    ''',

    # 忘记密码页面
    'forgot_password.html': '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>数独游戏 - 重置密码</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <style>
            * { margin: 0; padding: 0; box-sizing: border-box; }
            body { font-family: "Arial", sans-serif; max-width: 450px; margin: 50px auto; padding: 0 20px; }
            h2 { text-align: center; margin-bottom: 30px; color: #333; }
            .form-group { margin-bottom: 20px; }
            label { display: block; margin-bottom: 8px; color: #666; font-weight: 500; }
            input { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 4px; font-size: 16px; }
            button { width: 100%; padding: 12px; background: #4CAF50; color: white; border: none; border-radius: 4px; font-size: 16px; cursor: pointer; }
            button:hover { background: #45a049; }
            .links { margin-top: 20px; text-align: center; }
            a { color: #4CAF50; text-decoration: none; }
            .flash { color: #f44336; text-align: center; margin-bottom: 20px; padding: 10px; border: 1px solid #fadbd8; border-radius: 4px; }
        </style>
    </head>
    <body>
        <h2>数独游戏 - 重置密码</h2>
        {% with messages = get_flashed_messages() %}
          {% if messages %}
            {% for msg in messages %}
              <div class="flash">{{ msg }}</div>
            {% endfor %}
          {% endif %}
        {% endwith %}
        <form method="post">
            <div class="form-group">
                <label>用户名</label>
                <input type="text" name="username" required placeholder="请输入需要重置的用户名">
            </div>
            <div class="form-group">
                <label>新密码</label>
                <input type="password" name="new_password" required placeholder="请输入新密码">
            </div>
            <button type="submit">重置密码</button>
        </form>
        <div class="links">
            <a href="/login">返回登录</a>
        </div>
    </body>
    </html>
    ''',

    # 首页
    'dashboard.html': '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>数独游戏 - 首页</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <style>
            * { margin: 0; padding: 0; box-sizing: border-box; }
            body { font-family: "Arial", sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
            .nav { display: flex; justify-content: space-between; align-items: center; margin-bottom: 30px; padding-bottom: 15px; border-bottom: 1px solid #eee; }
            .nav a { color: #333; text-decoration: none; margin-right: 20px; font-size: 16px; }
            .nav a:hover, .nav a.active { color: #4CAF50; }
            .nav .user-info { color: #666; }
            h2 { color: #333; margin-bottom: 25px; }
            .level-list { display: grid; gap: 20px; }
            .level-card { border: 1px solid #eee; border-radius: 8px; padding: 20px; transition: box-shadow 0.3s; }
            .level-card:hover { box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
            .level-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; }
            .level-name { font-size: 18px; color: #333; margin: 0; }
            .difficulty { padding: 4px 8px; border-radius: 4px; font-size: 14px; color: white; }
            .difficulty.easy { background: #4CAF50; }
            .difficulty.medium { background: #ff9800; }
            .difficulty.hard { background: #f44336; }
            .level-actions a { display: inline-block; margin-right: 15px; color: #4CAF50; text-decoration: none; }
            .level-record { margin-top: 10px; font-size: 14px; color: #666; }
        </style>
    </head>
    <body>
        <div class="nav">
            <div>
                <a href="/dashboard" class="active">首页</a>
                <a href="/user/profile">个人中心</a>
                {% if session.is_admin %}
                <a href="/admin">管理员中心</a>
                {% endif %}
                <a href="/logout">退出登录</a>
            </div>
            <div class="user-info">欢迎，{{ session.username }}</div>
        </div>

        <h2>数独关卡列表</h2>
        <div class="level-list">
            {% for level in levels %}
            <div class="level-card">
                <div class="level-header">
                    <h3 class="level-name">{{ level.level_name }}</h3>
                    <span class="difficulty {{ level.difficulty }}">{{ level.difficulty }}</span>
                </div>
                <div class="level-actions">
                    <a href="/play/{{ level.id }}">开始游戏</a>
                    <a href="/ranking/{{ level.id }}">查看排名</a>
                </div>
                {% for record in user_records if record.level_id == level.id %}
                <div class="level-record">你的最佳成绩: {{ record.time_spent|format_time }}</div>
                {% endfor %}
            </div>
            {% else %}
            <div class="level-card">暂无关卡，请联系管理员添加</div>
            {% endfor %}
        </div>
    </body>
    </html>
    ''',

    # 游戏页面（修复循环变量引用错误）
    'play.html': '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>数独游戏 - {{ level.level_name }}</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <style>
            * { margin: 0; padding: 0; box-sizing: border-box; }
            body { font-family: "Arial", sans-serif; max-width: 600px; margin: 0 auto; padding: 20px; }
            h2 { color: #333; margin-bottom: 15px; text-align: center; }
            .difficulty-tag { font-size: 14px; color: #666; text-align: center; margin-bottom: 20px; }
            .timer { font-size: 24px; text-align: center; margin-bottom: 20px; color: #333; font-weight: bold; }
            /* 数独网格样式（关键：3x3宫格分隔） */
            .sudoku-grid { display: grid; grid-template-columns: repeat(9, 1fr); gap: 1px; border: 2px solid #333; margin-bottom: 20px; }
            .sudoku-cell { aspect-ratio: 1; border: 1px solid #ddd; display: flex; align-items: center; justify-content: center; }
            /* 3x3宫格分隔线（加粗） */
            .sudoku-cell:nth-child(3n) { border-right: 2px solid #333; }
            .sudoku-cell:nth-child(9n) { border-right: 1px solid #ddd; }
            .sudoku-grid > div:nth-child(n+19):nth-child(-n+27),
            .sudoku-grid > div:nth-child(n+46):nth-child(-n+54) { border-bottom: 2px solid #333; }
            .sudoku-cell input { width: 100%; height: 100%; border: none; text-align: center; font-size: 18px; outline: none; }
            .sudoku-cell input:disabled { background: #f5f5f5; font-weight: bold; color: #333; }
            .controls { display: flex; justify-content: center; gap: 15px; margin-bottom: 20px; }
            button { padding: 10px 20px; background: #4CAF50; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 16px; }
            button:hover { background: #45a049; }
            .message { text-align: center; padding: 10px; border-radius: 4px; margin-bottom: 20px; }
            .message.success { background: #dff0d8; color: #3c763d; }
            .message.error { background: #f2dede; color: #a94442; }
            .back-link { text-align: center; }
            .back-link a { color: #4CAF50; text-decoration: none; }
        </style>
    </head>
    <body>
        <h2>{{ level.level_name }}</h2>
        <div class="difficulty-tag">难度：{{ level.difficulty }}</div>
        <div class="timer" id="timer">00:00</div>
        
        <!-- 数独网格 -->
        <div class="sudoku-grid">
            {% for i in range(9) %}
                {% for j in range(9) %}
                <div class="sudoku-cell">
                    {% if grid[i][j] != '0' %}
                        <input type="text" value="{{ grid[i][j] }}" disabled maxlength="1">
                    {% else %}
                        <input type="text" maxlength="1" data-row="{{ i }}" data-col="{{ j }}" 
                               oninput="this.value = this.value.replace(/[^1-9]/g, '')">
                    {% endif %}
                </div>
                {% endfor %}
            {% endfor %}
        </div>
        
        <div class="controls">
            <button onclick="submitSolution()">提交答案</button>
            <button onclick="resetBoard()">重置</button>
            <button onclick="window.location.href='/dashboard'">返回首页</button>
        </div>
        
        <div class="message" id="message"></div>

        <script>
            let seconds = 0;  // 总耗时（秒）
            let timerInterval = null;

            // 页面加载时启动计时器
            window.onload = function() {
                timerInterval = setInterval(updateTimer, 1000);
            };

            // 更新计时器显示（格式：MM:SS）
            function updateTimer() {
                seconds++;
                const minutes = Math.floor(seconds / 60).toString().padStart(2, '0');
                const secs = (seconds % 60).toString().padStart(2, '0');
                document.getElementById('timer').textContent = `${minutes}:${secs}`;
            }

            // 停止计时器
            function stopTimer() {
                clearInterval(timerInterval);
            }

            // 提交答案到服务器验证
            function submitSolution() {
                const solution = [];
                // 确保正确收集所有单元格值（包括固定数字）
                for (let i = 0; i < 9; i++) {
                    for (let j = 0; j < 9; j++) {
                        // 优先查找可编辑单元格
                        const editableCell = document.querySelector(`input[data-row="${i}"][data-col="${j}"]`);
                        if (editableCell) {
                            // 空值或非数字视为0
                            const value = editableCell.value.trim() || '0';
                            solution.push(value);
                        } else {
                            // 固定数字（从禁用的输入框中获取）
                            const fixedCells = document.querySelectorAll(`.sudoku-grid > div:nth-child(${i*9 + j + 1}) input:disabled`);
                            if (fixedCells.length > 0) {
                                solution.push(fixedCells[0].value);
                            } else {
                                // 异常情况处理
                                solution.push('0');
                                console.error(`未找到单元格 (${i}, ${j}) 的值`);
                            }
                        }
                    }
                }

                // 检查是否有未填项（0）
                if (solution.includes('0')) {
                    showMessage('请填写所有空白单元格', 'error');
                    return;
                }

                // 发送请求（增加错误捕获和详细日志）
                fetch('/submit/{{ level.id }}', {
                    method: 'POST',
                    headers: { 
                        'Content-Type': 'application/json',
                        'X-Requested-With': 'XMLHttpRequest' // 标识AJAX请求
                    },
                    body: JSON.stringify({
                        solution: solution.join(''),
                        time_spent: seconds
                    })
                })
                .then(response => {
                    // 检查HTTP响应状态
                    if (!response.ok) {
                        throw new Error(`HTTP错误：${response.status}`);
                    }
                    return response.json();
                })
                .then(data => {
                    showMessage(data.message, data.success ? 'success' : 'error');
                    if (data.success) {
                        stopTimer();
                        // 禁用所有输入框
                        document.querySelectorAll('input:not([disabled])').forEach(input => {
                            input.disabled = true;
                        });
                    }
                })
                .catch(err => {
                    showMessage(`提交失败：${err.message}`, 'error');
                    console.error('提交错误详情：', err);
                });
            }

            // 重置棋盘（清空输入、重置计时器）
            function resetBoard() {
                document.querySelectorAll('input:not([disabled])').forEach(input => {
                    input.value = '';
                });
                showMessage('');
                // 重置计时器
                stopTimer();
                seconds = 0;
                document.getElementById('timer').textContent = '00:00';
                timerInterval = setInterval(updateTimer, 1000);
            }

            // 显示消息提示
            function showMessage(text, type) {
                const msgEl = document.getElementById('message');
                msgEl.textContent = text;
                msgEl.className = 'message';
                if (type) msgEl.classList.add(type);
            }

            // 页面关闭前停止计时器
            window.onbeforeunload = stopTimer;
        </script>
    </body>
    </html>
    ''',

    # 排名页面
    'ranking.html': '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>数独游戏 - {{ level.level_name }} 排名</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <style>
            * { margin: 0; padding: 0; box-sizing: border-box; }
            body { font-family: "Arial", sans-serif; max-width: 600px; margin: 0 auto; padding: 20px; }
            h2 { color: #333; margin-bottom: 10px; text-align: center; }
            .difficulty-tag { font-size: 14px; color: #666; text-align: center; margin-bottom: 20px; }
            .ranking-table { width: 100%; border-collapse: collapse; margin-bottom: 20px; }
            .ranking-table th, .ranking-table td { border: 1px solid #ddd; padding: 12px; text-align: left; }
            .ranking-table th { background: #f5f5f5; color: #333; }
            .ranking-table tr:nth-child(even) { background: #f9f9f9; }
            .ranking-table tr.highlight { background: #dff0d8; }  /* 高亮当前用户 */
            .user-status { text-align: center; margin-bottom: 20px; padding: 10px; border-radius: 4px; }
            .back-links { text-align: center; }
            .back-links a { display: inline-block; margin: 0 10px; color: #4CAF50; text-decoration: none; }
        </style>
    </head>
    <body>
        <h2>{{ level.level_name }} 排名榜</h2>
        <div class="difficulty-tag">难度：{{ level.difficulty }}</div>

        <!-- 排名表格 -->
        <table class="ranking-table">
            <thead>
                <tr>
                    <th>排名</th>
                    <th>用户名</th>
                    <th>用时</th>
                </tr>
            </thead>
            <tbody>
                {% if records %}
                    {% for record in records %}
                    <tr {% if session.username == record.username %}class="highlight"{% endif %}>
                        <td>{{ loop.index }}</td>
                        <td>{{ record.username }}</td>
                        <td>{{ record.time_spent|format_time }}</td>
                    </tr>
                    {% endfor %}
                {% else %}
                    <tr>
                        <td colspan="3" style="text-align: center;">暂无解题记录</td>
                    </tr>
                {% endif %}
            </tbody>
        </table>

        <!-- 当前用户状态 -->
        <div class="user-status">
            {% if user_record %}
                {% set user_rank = 0 %}
                {% for record in records %}
                    {% if session.username == record.username %}
                        {% set user_rank = loop.index %}
                    {% endif %}
                {% endfor %}
                <p>你的排名：{{ user_rank if user_rank > 0 else '未上榜' }}</p>
                <p>你的用时：{{ user_record.time_spent|format_time }}</p>
            {% else %}
                <p>你尚未完成这一关卡，快去挑战吧！</p>
            {% endif %}
        </div>

        <div class="back-links">
            <a href="/play/{{ level.id }}">继续游戏</a>
            <a href="/dashboard">返回首页</a>
        </div>

        <!-- 实时刷新排名（1分钟一次） -->
        <script>
            setInterval(function() {
                fetch('/get_live_ranking/{{ level.id }}')
                    .then(res => res.json())
                    .then(data => {
                        if (data.ranking.length === 0) return;
                        const tbody = document.querySelector('.ranking-table tbody');
                        // 清空现有数据（保留表头）
                        tbody.innerHTML = '';
                        // 填充新排名
                        data.ranking.forEach((item, index) => {
                            const tr = document.createElement('tr');
                            // 高亮当前用户
                            if (item.username === '{{ session.username }}') {
                                tr.classList.add('highlight');
                            }
                            tr.innerHTML = `
                                <td>${index + 1}</td>
                                <td>${item.username}</td>
                                <td>${formatTime(item.time_spent)}</td>
                            `;
                            tbody.appendChild(tr);
                        });
                        // 更新用户自身排名
                        if (data.user_rank) {
                            document.querySelector('.user-status p:first-child').textContent = `你的排名：${data.user_rank}`;
                            document.querySelector('.user-status p:last-child').textContent = `你的用时：${formatTime(data.user_time)}`;
                        }
                    })
                    .catch(err => console.error('排名刷新失败：', err));
            }, 60000);  // 60秒刷新一次

            // 时间格式化函数（秒转MM:SS）
            function formatTime(seconds) {
                const minutes = Math.floor(seconds / 60).toString().padStart(2, '0');
                const secs = (seconds % 60).toString().padStart(2, '0');
                return `${minutes}:${secs}`;
            }
        </script>
    </body>
    </html>
    ''',

    # 管理员中心
    'admin_dashboard.html': '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>数独游戏 - 管理员中心</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <style>
            * { margin: 0; padding: 0; box-sizing: border-box; }
            body { font-family: "Arial", sans-serif; max-width: 1000px; margin: 0 auto; padding: 20px; }
            .nav { display: flex; justify-content: space-between; align-items: center; margin-bottom: 30px; padding-bottom: 15px; border-bottom: 1px solid #eee; }
            .nav a { color: #333; text-decoration: none; margin-right: 20px; font-size: 16px; }
            .nav a:hover { color: #4CAF50; }
            .nav .user-info { color: #666; }
            .section { margin-bottom: 40px; }
            h2 { color: #333; margin-bottom: 20px; }
            h3 { color: #444; margin-bottom: 15px; }
            .add-btn { display: inline-block; margin-bottom: 15px; padding: 8px 15px; background: #2196F3; color: white; text-decoration: none; border-radius: 4px; }
            table { width: 100%; border-collapse: collapse; margin-bottom: 20px; }
            th, td { border: 1px solid #ddd; padding: 12px; text-align: left; }
            th { background: #f5f5f5; color: #333; }
            .action-btn { padding: 6px 10px; border: none; border-radius: 4px; cursor: pointer; font-size: 14px; }
            .btn-enable { background: #4CAF50; color: white; }
            .btn-disable { background: #f44336; color: white; }
            .btn-edit { background: #ff9800; color: white; }
            .btn-delete { background: #f44336; color: white; }
            .status-active { color: #4CAF50; }
            .status-inactive { color: #f44336; }
        </style>
    </head>
    <body>
        <div class="nav">
            <div>
                <a href="/dashboard">返回首页</a>
                <a href="/user/profile">个人中心</a>
                <a href="/logout">退出登录</a>
            </div>
            <div class="user-info">管理员：{{ session.username }}</div>
        </div>

        <!-- 用户管理 -->
        <div class="section">
            <h2>用户管理</h2>
            <table>
                <thead>
                    <tr>
                        <th>ID</th>
                        <th>用户名</th>
                        <th>注册时间</th>
                        <th>状态</th>
                        <th>操作</th>
                    </tr>
                </thead>
                <tbody>
                    {% for user in users %}
                    <tr>
                        <td>{{ user.id }}</td>
                        <td>
                            {{ user.username }}
                            {% if user.is_admin %}
                            <span style="color:#2196F3; font-size:12px;">(管理员)</span>
                            {% endif %}
                        </td>
                        <td>{{ user.created_at.strftime('%Y-%m-%d %H:%M') }}</td>
                        <td>
                            <span class="{{ 'status-active' if user.is_active else 'status-inactive' }}">
                                {{ '启用' if user.is_active else '禁用' }}
                            </span>
                        </td>
                        <td>
                            {% if not user.is_admin %}  <!-- 不能操作管理员账号 -->
                            <a href="/admin/toggle_user/{{ user.id }}">
                                <button class="action-btn {{ 'btn-disable' if user.is_active else 'btn-enable' }}">
                                    {{ '禁用' if user.is_active else '启用' }}
                                </button>
                            </a>
                            {% endif %}
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>

        <!-- 关卡管理 -->
        <div class="section">
            <h2>关卡管理</h2>
            <a href="/admin/add_level" class="add-btn">添加新关卡</a>
            <table>
                <thead>
                    <tr>
                        <th>ID</th>
                        <th>关卡名称</th>
                        <th>难度</th>
                        <th>创建时间</th>
                        <th>操作</th>
                    </tr>
                </thead>
                <tbody>
                    {% for level in levels %}
                    <tr>
                        <td>{{ level.id }}</td>
                        <td>{{ level.level_name }}</td>
                        <td>{{ level.difficulty }}</td>
                        <td>{{ level.created_at.strftime('%Y-%m-%d %H:%M') }}</td>
                        <td>
                            <a href="/admin/edit_level/{{ level.id }}"><button class="action-btn btn-edit">编辑</button></a>
                            <a href="/admin/delete_level/{{ level.id }}" onclick="return confirm('确定删除？相关解题记录也会被删除！')">
                                <button class="action-btn btn-delete">删除</button>
                            </a>
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </body>
    </html>
    ''',

    # 添加关卡页面
    'add_level.html': '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>数独游戏 - 添加关卡</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <style>
            * { margin: 0; padding: 0; box-sizing: border-box; }
            body { font-family: "Arial", sans-serif; max-width: 450px; margin: 50px auto; padding: 0 20px; }
            h2 { text-align: center; margin-bottom: 30px; color: #333; }
            .form-group { margin-bottom: 20px; }
            label { display: block; margin-bottom: 8px; color: #666; font-weight: 500; }
            input, select { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 4px; font-size: 16px; }
            button { width: 100%; padding: 12px; background: #4CAF50; color: white; border: none; border-radius: 4px; font-size: 16px; cursor: pointer; }
            button:hover { background: #45a049; }
            .links { margin-top: 20px; text-align: center; }
            a { color: #4CAF50; text-decoration: none; }
        </style>
    </head>
    <body>
        <h2>添加新数独关卡</h2>
        <form method="post">
            <div class="form-group">
                <label>关卡名称</label>
                <input type="text" name="level_name" required placeholder="例如：简单关卡 - 第1关">
            </div>
            <div class="form-group">
                <label>难度等级</label>
                <select name="difficulty" required>
                    <option value="easy">简单（仅移除30个数字）</option>
                    <option value="medium" selected>中等（移除40个数字）</option>
                    <option value="hard">困难（移除50个数字）</option>
                </select>
            </div>
            <button type="submit">创建关卡</button>
        </form>
        <div class="links">
            <a href="/admin">返回管理员中心</a>
        </div>
    </body>
    </html>
    ''',

    # 编辑关卡页面
    'edit_level.html': '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>数独游戏 - 编辑关卡</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <style>
            * { margin: 0; padding: 0; box-sizing: border-box; }
            body { font-family: "Arial", sans-serif; max-width: 450px; margin: 50px auto; padding: 0 20px; }
            h2 { text-align: center; margin-bottom: 30px; color: #333; }
            .form-group { margin-bottom: 20px; }
            label { display: block; margin-bottom: 8px; color: #666; font-weight: 500; }
            input, select { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 4px; font-size: 16px; }
            .checkbox-group { display: flex; align-items: center; }
            .checkbox-group input { width: auto; margin-right: 10px; }
            button { width: 100%; padding: 12px; background: #4CAF50; color: white; border: none; border-radius: 4px; font-size: 16px; cursor: pointer; }
            button:hover { background: #45a049; }
            .links { margin-top: 20px; text-align: center; }
            a { color: #4CAF50; text-decoration: none; }
        </style>
    </head>
    <body>
        <h2>编辑关卡：{{ level.level_name }}</h2>
        <form method="post">
            <div class="form-group">
                <label>关卡名称</label>
                <input type="text" name="level_name" value="{{ level.level_name }}" required>
            </div>
            <div class="form-group">
                <label>难度等级</label>
                <select name="difficulty" required>
                    <option value="easy" {% if level.difficulty == 'easy' %}selected{% endif %}>简单（仅移除30个数字）</option>
                    <option value="medium" {% if level.difficulty == 'medium' %}selected{% endif %}>中等（移除40个数字）</option>
                    <option value="hard" {% if level.difficulty == 'hard' %}selected{% endif %}>困难（移除50个数字）</option>
                </select>
            </div>
            <div class="form-group checkbox-group">
                <input type="checkbox" name="regenerate" id="regenerate" value="1">
                <label for="regenerate">重新生成该难度的数独题目（原题目将被覆盖）</label>
            </div>
            <button type="submit">保存修改</button>
        </form>
        <div class="links">
            <a href="/admin">返回管理员中心</a>
        </div>
    </body>
    </html>
    ''',

    # 个人中心页面（修复'sudoku_level'访问错误）
    'user_profile.html': '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>数独游戏 - 个人中心</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <style>
            * { margin: 0; padding: 0; box-sizing: border-box; }
            body { font-family: "Arial", sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
            .nav { display: flex; justify-content: space-between; align-items: center; margin-bottom: 30px; padding-bottom: 15px; border-bottom: 1px solid #eee; }
            .nav a { color: #333; text-decoration: none; margin-right: 20px; font-size: 16px; }
            .nav a:hover { color: #4CAF50; }
            .section { margin-bottom: 40px; padding: 20px; border: 1px solid #eee; border-radius: 8px; }
            h2 { color: #333; margin-bottom: 30px; text-align: center; }
            h3 { color: #444; margin-bottom: 20px; padding-bottom: 10px; border-bottom: 1px solid #eee; }
            .profile-info { line-height: 2; color: #666; }
            .password-form { max-width: 450px; margin: 0 auto; }
            .form-group { margin-bottom: 20px; }
            label { display: block; margin-bottom: 8px; color: #666; font-weight: 500; }
            input { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 4px; font-size: 16px; }
            button { width: 100%; padding: 12px; background: #4CAF50; color: white; border: none; border-radius: 4px; font-size: 16px; cursor: pointer; }
            button:hover { background: #45a049; }
            .message { text-align: center; margin-top: 15px; padding: 10px; border-radius: 4px; }
            .message.success { background: #dff0d8; color: #3c763d; }
            .message.error { background: #f2dede; color: #a94442; }
            .record-table { width: 100%; border-collapse: collapse; margin-top: 20px; }
            .record-table th, .record-table td { border: 1px solid #ddd; padding: 12px; text-align: left; }
            .record-table th { background: #f5f5f5; color: #333; }
            .record-table tr:nth-child(even) { background: #f9f9f9; }
            .record-table a { color: #4CAF50; text-decoration: none; }
        </style>
    </head>
    <body>
        <h2>个人中心</h2>
        <div class="nav">
            <div>
                <a href="/dashboard">返回首页</a>
                <a href="/logout">退出登录</a>
            </div>
        </div>

        <!-- 个人信息 -->
        <div class="section">
            <h3>个人信息</h3>
            <div class="profile-info">
                <p><strong>用户名：</strong>{{ user.username }}</p>
                <p><strong>注册时间：</strong>{{ user.created_at.strftime('%Y-%m-%d %H:%M') }}</p>
                <p><strong>账号状态：</strong>{{ '正常' if user.is_active else '已禁用' }}</p>
            </div>
        </div>

        <!-- 修改密码 -->
        <div class="section">
            <h3>修改密码</h3>
            <form class="password-form" id="passwordForm">
                <div class="form-group">
                    <label>当前密码</label>
                    <input type="password" id="currentPwd" required placeholder="请输入当前密码">
                </div>
                <div class="form-group">
                    <label>新密码</label>
                    <input type="password" id="newPwd" required placeholder="请输入新密码">
                </div>
                <button type="submit">确认修改</button>
            </form>
            <div class="message" id="pwdMessage"></div>
        </div>

        <!-- 解题记录 -->
        <div class="section">
            <h3>我的解题记录</h3>
            <table class="record-table">
                <thead>
                    <tr>
                        <th>关卡名称</th>
                        <th>难度</th>
                        <th>用时</th>
                        <th>完成时间</th>
                        <th>操作</th>
                    </tr>
                </thead>
                <tbody>
                    {% if records %}
                        {% for record in records %}
                        <tr>
                            <td>{{ record.sudoku_level.level_name }}</td>  <!-- 正确访问关联对象 -->
                            <td>{{ record.sudoku_level.difficulty }}</td>
                            <td>{{ record.time_spent|format_time }}</td>
                            <td>{{ record.solved_at.strftime('%Y-%m-%d %H:%M') }}</td>
                            <td><a href="/play/{{ record.sudoku_level.id }}">再玩一次</a></td>
                        </tr>
                        {% endfor %}
                    {% else %}
                        <tr>
                            <td colspan="5" style="text-align: center;">暂无解题记录，快去挑战关卡吧！</td>
                        </tr>
                    {% endif %}
                </tbody>
            </table>
        </div>

        <script>
            // 处理密码修改
            document.getElementById('passwordForm').addEventListener('submit', function(e) {
                e.preventDefault();
                const currentPwd = document.getElementById('currentPwd').value;
                const newPwd = document.getElementById('newPwd').value;
                const msgEl = document.getElementById('pwdMessage');

                fetch('/user/change_password', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        current_password: currentPwd,
                        new_password: newPwd
                    })
                })
                .then(res => res.json())
                .then(data => {
                    msgEl.textContent = data.message;
                    msgEl.className = 'message';
                    if (data.success) {
                        msgEl.classList.add('success');
                        this.reset();  // 清空表单
                    } else {
                        msgEl.classList.add('error');
                    }
                })
                .catch(err => {
                    msgEl.textContent = '修改失败，请重试';
                    msgEl.className = 'message error';
                    console.error(err);
                });
            });
        </script>
    </body>
    </html>
    ''',

    # 二维码登录页面
    'qr_login.html': '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>数独游戏 - 二维码登录</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <style>
            * { margin: 0; padding: 0; box-sizing: border-box; }
            body { font-family: "Arial", sans-serif; text-align: center; padding: 20px; }
            h2 { color: #333; margin-bottom: 20px; }
            .tip { color: #666; margin-bottom: 30px; }
            .qr-container { display: inline-block; padding: 15px; background: white; box-shadow: 0 2px 10px rgba(0,0,0,0.1); margin-bottom: 20px; }
            .qr-container img { max-width: 250px; width: 100%; }
            .countdown { color: #f44336; font-size: 18px; margin-bottom: 30px; }
            .links { margin-top: 20px; }
            a { color: #4CAF50; text-decoration: none; }
        </style>
    </head>
    <body>
        <h2>二维码登录</h2>
        <p class="tip">请使用已登录的移动端扫描下方二维码（60秒内有效）</p>
        
        <div class="qr-container">
            <img src="/qr_code/{{ token }}" alt="数独游戏登录二维码" id="qrImg">
        </div>
        
        <div class="countdown">
            二维码将在 <span id="count">60</span> 秒后过期
        </div>
        
        <div class="links">
            <a href="/login">返回密码登录</a>
        </div>

        <script>
            let countdown = 60;
            const token = '{{ token }}';
            const countEl = document.getElementById('count');
            const qrImg = document.getElementById('qrImg');

            // 倒计时逻辑
            const countInterval = setInterval(function() {
                countdown--;
                countEl.textContent = countdown;
                if (countdown <= 0) {
                    clearInterval(countInterval);
                    // 二维码过期，刷新页面获取新二维码
                    window.location.reload();
                }
            }, 1000);

            // 检查登录状态（每2秒一次）
            const checkInterval = setInterval(function() {
                fetch(`/check_qr_login/${token}`)
                    .then(res => res.json())
                    .then(data => {
                        if (data.success) {
                            // 登录成功，跳转到首页
                            window.location.href = data.redirect;
                        } else if (data.expired) {
                            // 二维码过期，刷新页面
                            clearInterval(checkInterval);
                            window.location.reload();
                        }
                    })
                    .catch(err => console.error('登录状态检查失败：', err));
            }, 60000);
        </script>
    </body>
    </html>
    ''',

    # 二维码登录结果页面
    'qr_login_result.html': '''
    <!DOCTYPE html>
    <html>
    <head>
        <title>登录确认</title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <style>
            * { margin: 0; padding: 0; box-sizing: border-box; }
            body { font-family: "Arial", sans-serif; text-align: center; padding: 50px 20px; }
            .result { font-size: 24px; margin-bottom: 20px; }
            .success { color: #4CAF50; }
            .error { color: #f44336; }
            .tip { color: #666; font-size: 16px; }
        </style>
    </head>
    <body>
        <div class="result {{ 'success' if success else 'error' }}">
            {{ message }}
        </div>
        <p class="tip">页面将在3秒后自动关闭</p>

        <script>
            // 3秒后关闭当前页面
            setTimeout(function() {
                window.close();
            }, 3000);
        </script>
    </body>
    </html>
    '''
}

# -------------------------- 路由定义 --------------------------
@app.route('/')
def index():
    """首页重定向：已登录→首页，未登录→登录页"""
    if 'user_id' in session:
        return redirect(url_for('dashboard'))
    return redirect(url_for('login'))

@app.route('/login', methods=['GET', 'POST'])
def login():
    """用户登录"""
    if request.method == 'POST':
        username = request.form.get('username', '').strip()
        password = request.form.get('password', '').strip()
        # 验证用户（按要求不加密，直接匹配）
        user = User.query.filter_by(
            username=username, 
            password=password,
            is_active=True  # 仅允许启用的账号登录
        ).first()
        if user:
            # 登录成功，设置会话
            session['user_id'] = user.id
            session['username'] = user.username
            session['is_admin'] = user.is_admin
            return redirect(url_for('dashboard'))
        else:
            flash('用户名/密码错误，或账号已被禁用')
    # GET请求：渲染登录页
    return render_template_string(templates['login.html'])

@app.route('/register', methods=['GET', 'POST'])
def register():
    """用户注册"""
    if request.method == 'POST':
        username = request.form.get('username', '').strip()
        password = request.form.get('password', '').strip()
        confirm_pwd = request.form.get('confirm_password', '').strip()
        # 验证输入
        if password != confirm_pwd:
            flash('两次密码输入不一致')
            return render_template_string(templates['register.html'])
        if User.query.filter_by(username=username).first():
            flash('用户名已存在')
            return render_template_string(templates['register.html'])
        # 创建新用户
        new_user = User(username=username, password=password)
        db.session.add(new_user)
        db.session.commit()
        flash('注册成功，请登录')
        return redirect(url_for('login'))
    # GET请求：渲染注册页
    return render_template_string(templates['register.html'])

@app.route('/forgot_password', methods=['GET', 'POST'])
def forgot_password():
    """密码重置"""
    if request.method == 'POST':
        username = request.form.get('username', '').strip()
        new_password = request.form.get('new_password', '').strip()
        # 查找用户
        user = User.query.filter_by(username=username).first()
        if user:
            # 重置密码（按要求不加密）
            user.password = new_password
            db.session.commit()
            flash('密码重置成功，请登录')
            return redirect(url_for('login'))
        else:
            flash('用户名不存在')
    # GET请求：渲染重置页
    return render_template_string(templates['forgot_password.html'])

@app.route('/dashboard')
def dashboard():
    """用户首页（关卡列表）"""
    if 'user_id' not in session:
        return redirect(url_for('login'))
    # 获取所有关卡和当前用户的解题记录
    levels = SudokuLevel.query.all()
    user_records = SolvedRecord.query.filter_by(user_id=session['user_id']).all()
    return render_template_string(templates['dashboard.html'], levels=levels, user_records=user_records)

@app.route('/play/<int:level_id>')
def play(level_id):
    """游戏页面"""
    if 'user_id' not in session:
        return redirect(url_for('login'))
    # 获取关卡信息（不存在则404）
    level = SudokuLevel.query.get_or_404(level_id)
    # 转换为二维网格（便于模板渲染）
    grid = [list(level.puzzle[i*9:(i+1)*9]) for i in range(9)]
    return render_template_string(templates['play.html'], level=level, grid=grid)

@app.route('/submit/<int:level_id>', methods=['POST'])
def submit(level_id):
    """提交答案验证（增强错误处理）"""
    try:
        if 'user_id' not in session:
            return jsonify({'success': False, 'message': '请先登录'}), 401
        
        # 获取并验证请求数据
        if not request.is_json:
            return jsonify({'success': False, 'message': '请求格式错误，需为JSON'}), 400
        
        data = request.get_json()
        solution = data.get('solution', '').strip()
        time_spent = data.get('time_spent', 0)
        
        # 验证答案格式（必须是81位数字）
        if len(solution) != 81 or not solution.isdigit():
            return jsonify({'success': False, 'message': '答案格式错误'}), 400
        
        # 验证关卡存在
        level = SudokuLevel.query.get(level_id)
        if not level:
            return jsonify({'success': False, 'message': '关卡不存在'}), 404
        
        # 验证数独答案
        if not validate_sudoku(level.puzzle, solution):
            return jsonify({'success': False, 'message': '答案不正确，请检查后重试'})
        
        # 处理解题记录
        record = SolvedRecord.query.filter_by(
            user_id=session['user_id'],
            level_id=level_id
        ).first()
        
        if record:
            if time_spent < record.time_spent:
                record.time_spent = time_spent
                record.solved_at = datetime.utcnow()
                db.session.commit()
        else:
            new_record = SolvedRecord(
                user_id=session['user_id'],
                level_id=level_id,
                time_spent=time_spent
            )
            db.session.add(new_record)
            db.session.commit()
        
        return jsonify({
            'success': True, 
            'message': f'恭喜！解题成功，用时{format_time(time_spent)}'
        })
    
    except Exception as e:
        # 记录详细错误日志
        db.session.rollback()  # 回滚数据库操作
        print(f"提交接口错误：{str(e)}")
        return jsonify({'success': False, 'message': f'服务器错误：{str(e)}'}), 500

@app.route('/ranking/<int:level_id>')
def ranking(level_id):
    """关卡排名页面"""
    if 'user_id' not in session:
        return redirect(url_for('login'))
    # 获取关卡和排名数据（按用时升序）
    level = SudokuLevel.query.get_or_404(level_id)
    records = SolvedRecord.query.filter_by(level_id=level_id)\
        .join(User)\
        .add_columns(User.username, SolvedRecord.time_spent)\
        .order_by(SolvedRecord.time_spent)\
        .all()
    # 获取当前用户的解题记录
    user_record = SolvedRecord.query.filter_by(
        user_id=session['user_id'],
        level_id=level_id
    ).first()
    return render_template_string(templates['ranking.html'], level=level, records=records, user_record=user_record)

@app.route('/admin')
def admin_dashboard():
    """管理员中心"""
    # 仅允许管理员访问
    if 'user_id' not in session or not session['is_admin']:
        return redirect(url_for('login'))
    # 获取所有用户和关卡
    users = User.query.all()
    levels = SudokuLevel.query.all()
    return render_template_string(templates['admin_dashboard.html'], users=users, levels=levels)

@app.route('/admin/toggle_user/<int:user_id>')
def toggle_user_status(user_id):
    """启用/禁用用户（仅管理员）"""
    if 'user_id' not in session or not session['is_admin']:
        return redirect(url_for('login'))
    # 不允许操作管理员账号
    user = User.query.get_or_404(user_id)
    if user.is_admin:
        return redirect(url_for('admin_dashboard'))
    # 切换状态
    user.is_active = not user.is_active
    db.session.commit()
    return redirect(url_for('admin_dashboard'))

@app.route('/admin/add_level', methods=['GET', 'POST'])
def add_level():
    """添加关卡（仅管理员）"""
    if 'user_id' not in session or not session['is_admin']:
        return redirect(url_for('login'))
    if request.method == 'POST':
        level_name = request.form.get('level_name', '').strip()
        difficulty = request.form.get('difficulty', 'medium')
        # 生成数独题目
        puzzle = generate_sudoku(difficulty)
        # 创建关卡
        new_level = SudokuLevel(
            level_name=level_name,
            difficulty=difficulty,
            puzzle=puzzle,
            created_by=session['user_id']
        )
        db.session.add(new_level)
        db.session.commit()
        return redirect(url_for('admin_dashboard'))
    # GET请求：渲染添加页
    return render_template_string(templates['add_level.html'])

@app.route('/admin/edit_level/<int:level_id>', methods=['GET', 'POST'])
def edit_level(level_id):
    """编辑关卡（仅管理员）"""
    if 'user_id' not in session or not session['is_admin']:
        return redirect(url_for('login'))
    level = SudokuLevel.query.get_or_404(level_id)
    if request.method == 'POST':
        # 更新关卡信息
        level.level_name = request.form.get('level_name', '').strip()
        level.difficulty = request.form.get('difficulty', 'medium')
        # 若选择重新生成题目
        if request.form.get('regenerate') == '1':
            level.puzzle = generate_sudoku(level.difficulty)
        db.session.commit()
        return redirect(url_for('admin_dashboard'))
    # GET请求：渲染编辑页
    return render_template_string(templates['edit_level.html'], level=level)

@app.route('/admin/delete_level/<int:level_id>')
def delete_level(level_id):
    """删除关卡（仅管理员，级联删除解题记录）"""
    if 'user_id' not in session or not session['is_admin']:
        return redirect(url_for('login'))
    # 级联删除：先删除关联的解题记录，再删除关卡
    SolvedRecord.query.filter_by(level_id=level_id).delete()
    level = SudokuLevel.query.get_or_404(level_id)
    db.session.delete(level)
    db.session.commit()
    return redirect(url_for('admin_dashboard'))

@app.route('/user/profile')
def user_profile():
    """个人中心（修复核心错误：正确查询关联对象）"""
    if 'user_id' not in session:
        return redirect(url_for('login'))
    
    # 正确查询：通过关联关系获取完整对象（而非Row对象）
    user = User.query.get(session['user_id'])
    # 直接查询SolvedRecord对象，利用模型中定义的relationship访问关联的关卡信息
    records = SolvedRecord.query.filter_by(user_id=session['user_id'])\
                .order_by(SolvedRecord.solved_at.desc())\
                .all()
    
    return render_template_string(templates['user_profile.html'], user=user, records=records)

@app.route('/user/change_password', methods=['POST'])
def change_password():
    """修改密码接口"""
    if 'user_id' not in session:
        return jsonify({'success': False, 'message': '请先登录'})
    
    data = request.get_json()
    current_pwd = data.get('current_password', '').strip()
    new_pwd = data.get('new_password', '').strip()
    
    # 验证当前密码
    user = User.query.get(session['user_id'])
    if user.password != current_pwd:
        return jsonify({'success': False, 'message': '当前密码不正确'})
    
    # 更新密码
    user.password = new_pwd
    db.session.commit()
    return jsonify({'success': True, 'message': '密码修改成功'})

@app.route('/qr_login')
def qr_login():
    """二维码登录页面（生成新二维码）"""
    if 'user_id' in session:
        return redirect(url_for('dashboard'))  # 已登录直接跳转
    
    # 生成唯一token并保存到数据库
    token = generate_token()
    qr_login = QRLogin(token=token, user_id=None, used=False)  # user_id=-1表示未确认
    db.session.add(qr_login)
    db.session.commit()
    
    return render_template_string(templates['qr_login.html'], token=token)

@app.route('/qr_code/<token>')
def get_qr_code(token):
    """生成二维码图片（仅对有效token生成）"""
    # 验证token有效性（未使用且未过期）
    qr_login = QRLogin.query.filter_by(token=token, used=False).first()
    if not qr_login or datetime.utcnow() - qr_login.created_at > timedelta(seconds=60):
        return '二维码已过期', 404
    
    # 生成二维码并返回
    qr_buffer = generate_qr_code(token, request)
    return send_file(qr_buffer, mimetype='image/png')

@app.route('/qr_login_confirm/<token>')
def qr_login_confirm(token):
    """移动端确认登录（需已登录状态）"""
    if 'user_id' not in session:
        # 未登录则跳转到登录页，登录后再回到确认页
        return redirect(url_for('login', next=url_for('qr_login_confirm', token=token)))
    
    # 验证token
    qr_login = QRLogin.query.filter_by(token=token, used=False).first()
    if not qr_login or datetime.utcnow() - qr_login.created_at > timedelta(seconds=60):
        return render_template_string(templates['qr_login_result.html'], 
                                    success=False, message='二维码已过期')
    
    # 标记为已使用并关联用户
    qr_login.user_id = session['user_id']
    qr_login.used = True
    db.session.commit()
    
    return render_template_string(templates['qr_login_result.html'], 
                                success=True, message='登录确认成功')

@app.route('/check_qr_login/<token>')
def check_qr_login(token):
    """轮询检查二维码登录状态"""
    qr_login = QRLogin.query.filter_by(token=token, used=True).first()
    if qr_login and datetime.utcnow() - qr_login.created_at <= timedelta(seconds=60):
        # 登录成功：删除记录并返回跳转地址
        user = User.query.get(qr_login.user_id)
        db.session.delete(qr_login)
        db.session.commit()
        
        # 设置会话
        session['user_id'] = user.id
        session['username'] = user.username
        session['is_admin'] = user.is_admin
        
        return jsonify({'success': True, 'redirect': url_for('dashboard')})
    elif not qr_login or datetime.utcnow() - qr_login.created_at > timedelta(seconds=60):
        # 二维码过期
        return jsonify({'success': False, 'expired': True})
    else:
        # 等待确认
        return jsonify({'success': False, 'expired': False})

@app.route('/get_live_ranking/<int:level_id>')
def get_live_ranking(level_id):
    """实时排名接口（供前端定时刷新）"""
    # 获取前10名排名
    records = SolvedRecord.query.filter_by(level_id=level_id)\
                .join(User)\
                .add_columns(User.username, SolvedRecord.time_spent)\
                .order_by(SolvedRecord.time_spent)\
                .limit(10)\
                .all()
    ranking_data = [{'username': r.username, 'time_spent': r.time_spent} for r in records]
    
    # 获取当前用户排名（若已登录）
    user_rank = None
    user_time = None
    if 'user_id' in session:
        user_record = SolvedRecord.query.filter_by(
            user_id=session['user_id'], 
            level_id=level_id
        ).first()
        if user_record:
            user_time = user_record.time_spent
            # 计算排名（比当前用户快的人数 + 1）
            user_rank = SolvedRecord.query.filter_by(level_id=level_id)\
                        .filter(SolvedRecord.time_spent < user_time)\
                        .count() + 1
    
    return jsonify({
        'ranking': ranking_data,
        'user_rank': user_rank,
        'user_time': user_time
    })

@app.route('/logout')
def logout():
    """退出登录（清空会话）"""
    session.clear()
    return redirect(url_for('login'))

# -------------------------- 自定义模板过滤器 --------------------------
@app.template_filter('format_time')
def format_time(seconds):
    """将秒数转换为 MM:SS 格式"""
    minutes, seconds = divmod(seconds, 60)
    return f"{minutes:02d}:{seconds:02d}"

# -------------------------- 启动应用 --------------------------
if __name__ == '__main__':
    # 初始化数据库（在应用上下文内执行）
    with app.app_context():
        init_database()
    # 启动服务器（debug模式便于开发）
    app.run(debug=True, host='0.0.0.0', port=5000)
