from flask import Flask, request, jsonify, render_template
from models import db, Todo
from dotenv import load_dotenv
import os

load_dotenv()

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = os.getenv('DATABASE_URL', 'sqlite:///todos.db')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db.init_app(app)

# Home route (UI)
@app.route('/')
def index():
    return render_template('index.html')


# Get all todos + Add new todo
@app.route('/todos', methods=['GET', 'POST'])
def todos():
    if request.method == 'POST':
        data = request.get_json()

        todo = Todo(title=data['title'])
        db.session.add(todo)
        db.session.commit()

        return jsonify(todo.to_dict()), 201

    todos = Todo.query.all()
    return jsonify([todo.to_dict() for todo in todos])


# Update + Delete todo
@app.route('/todos/<int:id>', methods=['PUT', 'DELETE'])
def todo_detail(id):
    todo = Todo.query.get_or_404(id)

    if request.method == 'PUT':
        data = request.get_json()

        # ✅ COMPLETE / UNDO
        todo.completed = data.get('completed', todo.completed)

        # ✅ EDIT TITLE (NEW)
        todo.title = data.get('title', todo.title)

        db.session.commit()

        return jsonify(todo.to_dict())

    if request.method == 'DELETE':
        db.session.delete(todo)
        db.session.commit()

        return '', 204


if __name__ == '__main__':
    with app.app_context():
        db.create_all()

    app.run(host='0.0.0.0', port=5000, debug=True)
