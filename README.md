# pythondeveloperinternship
from flask import Flask, render_template, request, redirect, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager, UserMixin, login_user, logout_user
from flask_socketio import SocketIO, emit
from datetime import datetime
import pandas as pd
import numpy as np

app = Flask(__name__)

app.config['SECRET_KEY'] = 'secret123'
app.config['SQLALCHEMY_DATABASE_URI'] = 'postgresql://postgres:password@localhost/taskdb'

db = SQLAlchemy(app)
socketio = SocketIO(app)

login_manager = LoginManager()
login_manager.init_app(app)

# ------------------ DATABASE MODELS ------------------

class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(100), unique=True)
    password = db.Column(db.String(100))

class Task(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200))
    description = db.Column(db.String(500))
    priority = db.Column(db.String(50))
    status = db.Column(db.String(50))
    created_date = db.Column(db.DateTime, default=datetime.utcnow)

# ------------------ AUTH ------------------

@app.route('/register', methods=['POST'])
def register():
    data = request.json

    user = User(
        username=data['username'],
        password=data['password']
    )

    db.session.add(user)
    db.session.commit()

    return jsonify({"message": "User Registered"})

@app.route('/login', methods=['POST'])
def login():
    data = request.json

    user = User.query.filter_by(
        username=data['username'],
        password=data['password']
    ).first()

    if user:
        login_user(user)
        return jsonify({"message": "Login Success"})

    return jsonify({"message": "Invalid Credentials"})

@app.route('/logout')
def logout():
    logout_user()
    return jsonify({"message": "Logged Out"})

# ------------------ TASK APIs ------------------

@app.route('/tasks', methods=['POST'])
def add_task():
    data = request.json

    task = Task(
        title=data['title'],
        description=data['description'],
        priority=data['priority'],
        status=data['status']
    )

    db.session.add(task)
    db.session.commit()

    socketio.emit('task_update', {'message': 'New Task Added'})

    return jsonify({"message": "Task Added"})

@app.route('/tasks', methods=['GET'])
def get_tasks():

    tasks = Task.query.all()

    task_list = []

    for task in tasks:
        task_list.append({
            "id": task.id,
            "title": task.title,
            "description": task.description,
            "priority": task.priority,
            "status": task.status,
            "created_date": task.created_date
        })

    return jsonify(task_list)

@app.route('/tasks/<int:id>', methods=['PUT'])
def update_task(id):

    task = Task.query.get(id)
    data = request.json

    task.title = data['title']
    task.description = data['description']
    task.priority = data['priority']
    task.status = data['status']

    db.session.commit()

    socketio.emit('task_update', {'message': 'Task Updated'})

    return jsonify({"message": "Task Updated"})

@app.route('/tasks/<int:id>', methods=['DELETE'])
def delete_task(id):

    task = Task.query.get(id)

    db.session.delete(task)
    db.session.commit()

    socketio.emit('task_update', {'message': 'Task Deleted'})

    return jsonify({"message": "Task Deleted"})

# ------------------ ANALYTICS ------------------

@app.route('/analytics')
def analytics():

    tasks = Task.query.all()

    data = []

    for task in tasks:
        data.append({
            "status": task.status
        })

    df = pd.DataFrame(data)

    total_tasks = len(df)

    completed_tasks = len(df[df['status'] == 'Completed'])

    pending_tasks = len(df[df['status'] == 'Pending'])

    completion_percentage = np.round(
        (completed_tasks / total_tasks) * 100, 2
    ) if total_tasks > 0 else 0

    return jsonify({
        "Total Tasks": total_tasks,
        "Completed Tasks": completed_tasks,
        "Pending Tasks": pending_tasks,
        "Completion %": float(completion_percentage)
    })

# ------------------ FRONTEND ------------------

@app.route('/')
def home():
    return render_template('index.html')

# ------------------ MAIN ------------------

if __name__ == '__main__':
    with app.app_context():
        db.create_all()

    socketio.run(app, debug=True)f
