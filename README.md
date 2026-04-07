import os
from flask import Flask, render_template, request, redirect, url_for, session
from flask_sqlalchemy import SQLAlchemy
from werkzeug.utils import secure_filename

app = Flask(__name__)
app.secret_key = "supersecretkey"
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///students.db'
app.config['UPLOAD_FOLDER'] = "static/uploads"
db = SQLAlchemy(app)

# Database Models
class Student(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(100))
    email = db.Column(db.String(100), unique=True)
    password = db.Column(db.String(100))
    bio = db.Column(db.String(200))
    age = db.Column(db.Integer)
    photo = db.Column(db.String(200), default="default.png")
    courses = db.relationship('Course', backref='student', lazy=True)
    posts = db.relationship('Post', backref='author', lazy=True)

class Course(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(100))
    student_id = db.Column(db.Integer, db.ForeignKey('student.id'))

class Post(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    content = db.Column(db.String(500))
    student_id = db.Column(db.Integer, db.ForeignKey('student.id'))
    replies = db.relationship('Reply', backref='post', lazy=True)

class Reply(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    content = db.Column(db.String(300))
    student_id = db.Column(db.Integer, db.ForeignKey('student.id'))
    post_id = db.Column(db.Integer, db.ForeignKey('post.id'))

# Routes
@app.route("/")
def home():
    return render_template("index.html")

@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        name = request.form["name"]
        email = request.form["email"]
        password = request.form["password"]
        age = request.form["age"]
        bio = request.form["bio"]

        photo = request.files["photo"]
        filename = secure_filename(photo.filename)
        photo.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))

        new_student = Student(name=name, email=email, password=password,
                              age=age, bio=bio, photo=filename)
        db.session.add(new_student)
        db.session.commit()
        return redirect(url_for("login"))
    return render_template("register.html")

@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        email = request.form["email"]
        password = request.form["password"]
        student = Student.query.filter_by(email=email, password=password).first()
        if student:
            session["student_id"] = student.id
            return redirect(url_for("dashboard"))
    return render_template("login.html")

@app.route("/dashboard")
def dashboard():
    if "student_id" in session:
        student = Student.query.get(session["student_id"])
        return render_template("dashboard.html", student=student)
    return redirect(url_for("login"))

@app.route("/enroll", methods=["POST"])
def enroll():
    if "student_id" in session:
        course_title = request.form["course"]
        new_course = Course(title=course_title, student_id=session["student_id"])
        db.session.add(new_course)
        db.session.commit()
        return redirect(url_for("dashboard"))
    return redirect(url_for("login"))

@app.route("/activities")
def activities():
    return render_template("activities.html")

@app.route("/games")
def games():
    return render_template("games.html")

@app.route("/past-papers")
def past_papers():
    return render_template("past_papers.html")

@app.route("/forum", methods=["GET", "POST"])
def forum():
    if "student_id" not in session:
        return redirect(url_for("login"))
    if request.method == "POST":
        content = request.form["content"]
        new_post = Post(content=content, student_id=session["student_id"])
        db.session.add(new_post)
        db.session.commit()
        return redirect(url_for("forum"))
    posts = Post.query.all()
    return render_template("forum.html", posts=posts)

@app.route("/reply/<int:post_id>", methods=["POST"])
def reply(post_id):
    if "student_id" not in session:
        return redirect(url_for("login"))
    content = request.form["content"]
    new_reply = Reply(content=content, student_id=session["student_id"], post_id=post_id)
    db.session.add(new_reply)
    db.session.commit()
    return redirect(url_for("forum"))

if __name__ == "__main__":
    db.create_all()
    app.run(debug=True)
