# main.py - ЭТО НАСТОЯЩИЙ СЕРВЕР, А НЕ ПРОСТО HTML!
import os
import random
import string
import hashlib
from flask import Flask, render_template, request, redirect, session, jsonify
from flask_socketio import SocketIO, emit
from flask_mail import Mail, Message
from datetime import datetime, timedelta
import sqlite3
import json

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key-here'

# ===== НАСТРОЙКА ПОЧТЫ (GMAIL) =====
app.config['MAIL_SERVER'] = 'smtp.gmail.com'
app.config['MAIL_PORT'] = 587
app.config['MAIL_USE_TLS'] = True
app.config['MAIL_USERNAME'] = 'вашапочта@gmail.com'  # ТВОЯ GMAIL
app.config['MAIL_PASSWORD'] = 'ваш пароль'  # ПАРОЛЬ ОТ GMAIL
mail = Mail(app)

# ===== WEBSOCKET ДЛЯ ЧАТА =====
socketio = SocketIO(app, cors_allowed_origins="*")


# ===== БАЗА ДАННЫХ =====
def init_db():
    conn = sqlite3.connect('users.db')
    c = conn.cursor()

    # Таблица пользователей
    c.execute('''CREATE TABLE IF NOT EXISTS users
                 (
                     id
                     INTEGER
                     PRIMARY
                     KEY
                     AUTOINCREMENT,
                     email
                     TEXT
                     UNIQUE,
                     username
                     TEXT
                     UNIQUE,
                     password
                     TEXT,
                     avatar
                     TEXT,
                     online
                     BOOLEAN
                     DEFAULT
                     0,
                     last_seen
                     TIMESTAMP
                 )''')

    # Таблица сообщений
    c.execute('''CREATE TABLE IF NOT EXISTS messages
                 (
                     id
                     INTEGER
                     PRIMARY
                     KEY
                     AUTOINCREMENT,
                     from_user
                     INTEGER,
                     to_user
                     INTEGER,
                     message
                     TEXT,
                     timestamp
                     TIMESTAMP,
                     read
                     BOOLEAN
                     DEFAULT
                     0
                 )''')

    # Таблица сессий верификации
    c.execute('''CREATE TABLE IF NOT EXISTS verify_codes
                 (
                     email
                     TEXT,
                     code
                     TEXT,
                     timestamp
                     TIMESTAMP
                 )''')

    conn.commit()
    conn.close()


init_db()


# ===== ВЕРИФИКАЦИЯ ПОЧТЫ =====
def generate_code():
    return ''.join(random.choices(string.digits, k=6))


def send_verification_email(email, code):
    msg = Message('Код верификации GreenChat',
                  sender=app.config['MAIL_USERNAME'],
                  recipients=[email])
    msg.body = f'''
    Здравствуйте!

    Ваш код подтверждения для GreenChat: {code}

    Код действителен в течение 10 минут.

    Если вы не запрашивали код, просто проигнорируйте это письмо.

    С уважением,
    Команда GreenChat
    '''
    msg.html = f'''
    <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; padding: 20px; background: #f0f2f5; border-radius: 10px;">
        <div style="background: #00a884; padding: 20px; border-radius: 10px 10px 0 0; text-align: center;">
            <h1 style="color: white; margin: 0;">GreenChat</h1>
        </div>
        <div style="background: white; padding: 30px; border-radius: 0 0 10px 10px;">
            <h2 style="color: #111b21;">Подтверждение email</h2>
            <p style="color: #667781; font-size: 16px;">Ваш код подтверждения:</p>
            <div style="background: #00a884; color: white; font-size: 36px; font-weight: bold; text-align: center; padding: 20px; border-radius: 10px; letter-spacing: 5px;">
                {code}
            </div>
            <p style="color: #667781; font-size: 14px; margin-top: 20px;">Код действителен 10 минут</p>
        </div>
    </div>
    '''
    mail.send(msg)


@app.route('/')
def index():
    return render_template('index.html')


@app.route('/send-code', methods=['POST'])
def send_code():
    email = request.json.get('email')

    # Проверяем email
    if '@' not in email or '.' not in email:
        return jsonify({'success': False, 'error': 'Неверный email'})

    # Генерируем код
    code = generate_code()

    # Сохраняем в БД
    conn = sqlite3.connect('users.db')
    c = conn.cursor()
    c.execute("DELETE FROM verify_codes WHERE email = ?", (email,))
    c.execute("INSERT INTO verify_codes (email, code, timestamp) VALUES (?, ?, ?)",
              (email, code, datetime.now()))
    conn.commit()
    conn.close()

    # Отправляем email
    try:
        send_verification_email(email, code)
        return jsonify({'success': True})
    except Exception as e:
        return jsonify({'success': False, 'error': str(e)})


@app.route('/verify-code', methods=['POST'])
def verify_code():
    email = request.json.get('email')
    code = request.json.get('code')

    conn = sqlite3.connect('users.db')
    c = conn.cursor()

    # Проверяем код
    c.execute("SELECT timestamp FROM verify_codes WHERE email = ? AND code = ?",
              (email, code))
    result = c.fetchone()

    if result:
        # Проверяем время (10 минут)
        sent_time = datetime.fromisoformat(result[0])
        if datetime.now() - sent_time < timedelta(minutes=10):
            # Создаем временную сессию
            session['verified_email'] = email
            conn.close()
            return jsonify({'success': True})

    conn.close()
    return jsonify({'success': False, 'error': 'Неверный код'})


@app.route('/register', methods=['POST'])
def register():
    if 'verified_email' not in session:
        return jsonify({'success': False, 'error': 'Сначала подтвердите email'})

    email = session['verified_email']
    username = request.json.get('username')
    password = request.json.get('password')

    # Хешируем пароль
    hashed = hashlib.sha256(password.encode()).hexdigest()

    conn = sqlite3.connect('users.db')
    c = conn.cursor()

    try:
        c.execute("INSERT INTO users (email, username, password) VALUES (?, ?, ?)",
                  (email, username, hashed))
        conn.commit()

        # Получаем ID пользователя
        user_id = c.lastrowid

        # Создаем сессию
        session['user_id'] = user_id
        session['username'] = username

        conn.close()
        return jsonify({'success': True, 'username': username})
    except sqlite3.IntegrityError:
        conn.close()
        return jsonify({'success': False, 'error': 'Имя пользователя занято'})


@app.route('/search-users', methods=['POST'])
def search_users():
    if 'user_id' not in session:
        return jsonify({'success': False, 'error': 'Не авторизован'})

    query = request.json.get('query')

    conn = sqlite3.connect('users.db')
    c = conn.cursor()
    c.execute("SELECT id, username FROM users WHERE username LIKE ? AND id != ?",
              (f'%{query}%', session['user_id']))
    users = [{'id': row[0], 'username': row[1]} for row in c.fetchall()]
    conn.close()

    return jsonify({'success': True, 'users': users})


@app.route('/send-message', methods=['POST'])
def send_message():
    if 'user_id' not in session:
        return jsonify({'success': False, 'error': 'Не авторизован'})

    to_user = request.json.get('to')
    message = request.json.get('message')

    conn = sqlite3.connect('users.db')
    c = conn.cursor()
    c.execute("INSERT INTO messages (from_user, to_user, message, timestamp) VALUES (?, ?, ?, ?)",
              (session['user_id'], to_user, message, datetime.now()))
    conn.commit()
    conn.close()

    # Отправляем через WebSocket
    socketio.emit('new_message', {
        'from': session['user_id'],
        'to': to_user,
        'message': message,
        'username': session['username']
    })

    return jsonify({'success': True})


@app.route('/get-messages', methods=['POST'])
def get_messages():
    if 'user_id' not in session:
        return jsonify({'success': False, 'error': 'Не авторизован'})

    with_user = request.json.get('user')

    conn = sqlite3.connect('users.db')
    c = conn.cursor()
    c.execute('''SELECT m.*, u.username
                 FROM messages m
                          JOIN users u ON m.from_user = u.id
                 WHERE (from_user = ? AND to_user = ?)
                    OR (from_user = ? AND to_user = ?)
                 ORDER BY timestamp''',
              (session['user_id'], with_user, with_user, session['user_id']))

    messages = [{
        'from': row[1],
        'to': row[2],
        'message': row[3],
        'timestamp': row[4],
        'username': row[6],
        'is_me': row[1] == session['user_id']
    } for row in c.fetchall()]

    conn.close()
    return jsonify({'success': True, 'messages': messages})


if __name__ == '__main__':
    socketio.run(app, debug=True, port=5000)
