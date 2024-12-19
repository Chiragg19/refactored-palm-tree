from flask import Flask, jsonify, request, abort
from typing import Dict, List, Optional
import uuid

app = Flask(__name__)

# In-memory storage
database = {
    "books": [],  # List of book records
    "members": []  # List of member records
}

# Token for simple authentication
auth_token = "simple-token-123"

# Models
def generate_id() -> str:
    return str(uuid.uuid4())

class Book:
    def __init__(self, title: str, author: str, published_year: int):
        self.id = generate_id()
        self.title = title
        self.author = author
        self.published_year = published_year

    def to_dict(self) -> Dict[str, str]:
        return {
            "id": self.id,
            "title": self.title,
            "author": self.author,
            "published_year": self.published_year
        }

class Member:
    def __init__(self, name: str, email: str):
        self.id = generate_id()
        self.name = name
        self.email = email

    def to_dict(self) -> Dict[str, str]:
        return {
            "id": self.id,
            "name": self.name,
            "email": self.email
        }

# Middleware for Token Authentication
def require_auth(func):
    def wrapper(*args, **kwargs):
        token = request.headers.get("Authorization")
        if token != f"Bearer {auth_token}":
            abort(401, "Unauthorized")
        return func(*args, **kwargs)
    wrapper.__name__ = func.__name__
    return wrapper

# CRUD for Books
@app.route('/books', methods=['POST'])
@require_auth
def create_book():
    data = request.json
    if not data or not all(key in data for key in ("title", "author", "published_year")):
        abort(400, "Invalid input data")
    book = Book(title=data['title'], author=data['author'], published_year=data['published_year'])
    database['books'].append(book)
    return jsonify(book.to_dict()), 201

@app.route('/books', methods=['GET'])
@require_auth
def get_books():
    title_filter = request.args.get('title')
    author_filter = request.args.get('author')
    page = int(request.args.get('page', 1))
    page_size = int(request.args.get('page_size', 5))

    books = database['books']
    if title_filter:
        books = [book for book in books if title_filter.lower() in book.title.lower()]
    if author_filter:
        books = [book for book in books if author_filter.lower() in book.author.lower()]

    start = (page - 1) * page_size
    end = start + page_size
    paginated_books = books[start:end]
    return jsonify([book.to_dict() for book in paginated_books])

@app.route('/books/<book_id>', methods=['GET'])
@require_auth
def get_book(book_id):
    book = next((book for book in database['books'] if book.id == book_id), None)
    if not book:
        abort(404, "Book not found")
    return jsonify(book.to_dict())

@app.route('/books/<book_id>', methods=['PUT'])
@require_auth
def update_book(book_id):
    data = request.json
    book = next((book for book in database['books'] if book.id == book_id), None)
    if not book:
        abort(404, "Book not found")

    if 'title' in data:
        book.title = data['title']
    if 'author' in data:
        book.author = data['author']
    if 'published_year' in data:
        book.published_year = data['published_year']

    return jsonify(book.to_dict())

@app.route('/books/<book_id>', methods=['DELETE'])
@require_auth
def delete_book(book_id):
    book = next((book for book in database['books'] if book.id == book_id), None)
    if not book:
        abort(404, "Book not found")

    database['books'].remove(book)
    return jsonify({"message": "Book deleted"}), 200

# CRUD for Members
@app.route('/members', methods=['POST'])
@require_auth
def create_member():
    data = request.json
    if not data or not all(key in data for key in ("name", "email")):
        abort(400, "Invalid input data")
    member = Member(name=data['name'], email=data['email'])
    database['members'].append(member)
    return jsonify(member.to_dict()), 201

@app.route('/members', methods=['GET'])
@require_auth
def get_members():
    return jsonify([member.to_dict() for member in database['members']])

@app.route('/members/<member_id>', methods=['GET'])
@require_auth
def get_member(member_id):
    member = next((member for member in database['members'] if member.id == member_id), None)
    if not member:
        abort(404, "Member not found")
    return jsonify(member.to_dict())

@app.route('/members/<member_id>', methods=['PUT'])
@require_auth
def update_member(member_id):
    data = request.json
    member = next((member for member in database['members'] if member.id == member_id), None)
    if not member:
        abort(404, "Member not found")

    if 'name' in data:
        member.name = data['name']
    if 'email' in data:
        member.email = data['email']

    return jsonify(member.to_dict())

@app.route('/members/<member_id>', methods=['DELETE'])
@require_auth
def delete_member(member_id):
    member = next((member for member in database['members'] if member.id == member_id), None)
    if not member:
        abort(404, "Member not found")

    database['members'].remove(member)
    return jsonify({"message": "Member deleted"}), 200

if __name__ == '__main__':
    app.run(debug=True)
