## run.py
```python
from app import app

if __name__ == '__main__':
    app.run(debug=True)
```
## database.py
```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash
from flask_login import UserMixin
from datetime import datetime
# create the extension
db = SQLAlchemy()
# create the app
app = Flask(__name__)
# configure the SQLite database, relative to the app instance folder
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///site.db"
# initialize the app with the extension
db.init_app(app)
class User(db.Model, UserMixin):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(20), unique=True, nullable=False)
    password = db.Column(db.String(120), nullable=False)

    def set_password(self, password):
        self.password = generate_password_hash(password)
        
    def check_password(self, password):
        return check_password_hash(self.password, password)
class Message(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    message = db.Column(db.String(255))


class BlogPost(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(100), nullable=False)
    content = db.Column(db.Text, nullable=False)
    image_url = db.Column(db.String(200), nullable=True)
    date_posted = db.Column(db.DateTime, nullable=False, default=datetime.utcnow)
    read_time = db.Column(db.String(50), nullable=True)
    
    def __repr__(self):
        return f"BlogPost('{self.title}', '{self.date_posted}')"

with app.app_context():
    db.create_all()    
```

## __init__.py
```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager
from flask_ckeditor import CKEditor

app = Flask(__name__)
ckeditor = CKEditor(app)
app.config['UPLOAD_FOLDER'] = 'app/static'  # Gantilah dengan lokasi penyimpanan yang sesuai
app.config['SECRET_KEY'] = 'mysecret'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///site.db'
db = SQLAlchemy(app)
login_manager = LoginManager(app)
login_manager.login_view = 'login'

from app import routes

```
## form.py
```python
from flask_wtf import FlaskForm
from wtforms import StringField, TextAreaField, FileField 
from flask_ckeditor import CKEditorField

class BlogPostForm(FlaskForm):
    title = StringField('Title')
    content = CKEditorField('Content')
    #scontent = TextAreaField('Content')
    image = FileField('Image')
    read_time = StringField('Read Time')

class EditBlogPostForm(FlaskForm):
    title = StringField('Title')
    content = CKEditorField('Content')
    image = FileField('Image')
    read_time = StringField('Read Time')

```
## models.py
```python
from app import db
from werkzeug.security import generate_password_hash, check_password_hash
from flask_login import UserMixin
from datetime import datetime

class User(db.Model, UserMixin):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(20), unique=True, nullable=False)
    password = db.Column(db.String(120), nullable=False)

    def set_password(self, password):
        self.password = generate_password_hash(password)
        
    def check_password(self, password):
        return check_password_hash(self.password, password)

class Message(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    message = db.Column(db.String(255))

class BlogPost(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(100), nullable=False)
    content = db.Column(db.Text, nullable=False)
    image_url = db.Column(db.String(200), nullable=True)
    date_posted = db.Column(db.DateTime, nullable=False, default=datetime.utcnow)
    read_time = db.Column(db.String(50), nullable=True)
    
    def __repr__(self):
        return f"BlogPost('{self.title}', '{self.date_posted}')"

```
## routes.py
```python
from flask import render_template, url_for, flash, redirect, request
from app import app, db, login_manager
from app.models import User, Message, BlogPost
from flask_login import login_user, current_user, logout_user, login_required

from app.forms import BlogPostForm, EditBlogPostForm
import os
from werkzeug.utils import secure_filename

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))


@app.route("/")

def main():
    posts = BlogPost.query.order_by(BlogPost.date_posted.desc()).all()
    # Mengekstrak intro dari setiap postingan dan menambahkannya ke data postingan
    for post in posts:
        post = post.content

    return render_template('main.html', posts=posts)

@app.route("/dashboard")
@login_required
def dashboard():
    messages = Message.query.all()
    return render_template('dashboard.html', messages=messages)

@app.route("/login", methods=['GET', 'POST'])
def login():
    messages = Message.query.all()
    if current_user.is_authenticated:
        return redirect(url_for('dashboard'))
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        user = User.query.filter_by(username=username).first()
        if user and user.check_password(password):
            login_user(user)
            return redirect(url_for('dashboard'))
        else:
            flash('Login Unsuccessful. Please check username and password', 'danger')
    return render_template('login.html', messages=messages)

@app.route("/logout")
def logout():
    logout_user()
    return redirect(url_for('login'))

@app.route("/register", methods=['GET', 'POST'])
def register():
    if current_user.is_authenticated:
        return redirect(url_for('dashboard'))
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        user = User.query.filter_by(username=username).first()
        if user:
            flash('Username already exists. Please choose a different one.', 'danger')
            return redirect(url_for('register'))
        new_user = User(username=username)
        new_user.set_password(password)
        db.session.add(new_user)
        db.session.commit()
        flash('Your account has been created! You are now able to log in.', 'success')
        return redirect(url_for('login'))
    return render_template('register.html')

@app.route('/create', methods=['POST'])
def create():
    new_message = request.form.get('new_message')
    if new_message:
        message = Message(message=new_message)
        db.session.add(message)
        db.session.commit()
    return redirect(url_for('dashboard'))

@app.route('/delete/<int:id>', methods=['POST'])
def delete(id):
    message = Message.query.get_or_404(id)
    db.session.delete(message)
    db.session.commit()
    return redirect(url_for('dashboard'))

@app.route('/post/<int:post_id>')
def post_detail(post_id):
    post = BlogPost.query.get_or_404(post_id)
    return render_template('post_detail.html', post=post)

@app.route('/post/new', methods=['GET', 'POST'])
def add_post():
    form = BlogPostForm()
    if request.method == 'POST':
        title = request.form.get('title')
        content = request.form.get('content')
        read_time = request.form.get('read_time')
        
        # Inisialisasi image_url dengan None sebagai default
        image_url = None

        # Tangani pengunggahan gambar
        if 'image' in request.files:
            image = request.files['image']
            if image.filename != '':
                # Pastikan direktori penyimpanan gambar ada (biasanya dalam folder 'static/uploads/images')
                upload_folder = os.path.join(app.config['UPLOAD_FOLDER'], 'images')
                os.makedirs(upload_folder, exist_ok=True)

                # Secure filename untuk menghindari masalah keamanan
                filename = secure_filename(image.filename)

                # Simpan gambar yang diunggah ke direktori
                image.save(os.path.join(upload_folder, filename))

                # Set nilai 'image_url' dengan nama file saja
                image_url = filename

        # Buat postingan dengan 'image_url' yang sesuai
        post = BlogPost(title=title, content=content, image_url=image_url, read_time=read_time)
        db.session.add(post)
        db.session.commit()
        
        flash('Post berhasil dibuat', 'success')
        return redirect(url_for('dashboard'))
    
    return render_template('add_post.html', form=form)

@app.route('/post/edit/<int:post_id>', methods=['GET', 'POST'])
@login_required
def edit_post(post_id):
    post = BlogPost.query.get_or_404(post_id)
    form = EditBlogPostForm()

    if request.method == 'POST' and form.validate():
        post.title = form.title.data
        post.content = form.content.data
        post.read_time = form.read_time.data

        # Handle image upload if present
        if 'image' in request.files:
            image = request.files['image']
            if image.filename != '':
                upload_folder = os.path.join(app.config['UPLOAD_FOLDER'], 'images')
                os.makedirs(upload_folder, exist_ok=True)
                filename = secure_filename(image.filename)
                image.save(os.path.join(upload_folder, filename))
                post.image_url = filename  # Update the image URL

        db.session.commit()
        flash('Post berhasil diperbarui', 'success')
        return redirect(url_for('post_detail', post_id=post.id))

    elif request.method == 'GET':
        form.title.data = post.title
        form.content.data = post.content
        form.read_time.data = post.read_time

    return render_template('edit_post.html', form=form, post=post)

@app.route('/post/delete/<int:post_id>', methods=['POST'])
@login_required
def delete_post(post_id):
    post = BlogPost.query.get_or_404(post_id)
    db.session.delete(post)
    db.session.commit()
    flash('Post berhasil dihapus', 'success')
    return redirect(url_for('edit_pos'))

@app.route('/edit_post')
@login_required
def edit_pos():
    posts = BlogPost.query.order_by(BlogPost.date_posted.desc()).all()
    return render_template('list_edit.html', posts=posts)
```
> download file templates dan static [disini](https://tracerstudy.stbapontianak.ac.id/completestatic.zip)
