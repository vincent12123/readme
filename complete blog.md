 ## Struktur Program 
```
flasklogin1
├── app
│   ├── models.py
│   ├── routes.py
│   ├── static
│   │   ├── assets
│   │   ├── css
│   │   └── img
│   ├── templates
│   │   ├── base.html
│   │   ├── basedashboard.html
│   │   ├── home.html
│   │   ├── index.html
│   │   ├── login.html
│   │   ├── main.html
│   │   ├── register.html
│   │   ├── routes.py
│   │   └── sidebar.html
│   ├── __init__.py
├── database.py
├── instance
│   └── site.db
└── run.py

```
Weighted Mean Grade didefinisikan sebagai:

$$
\text{Weighted Mean Grade} = \frac{\sum (\text{Nilai Mata Kuliah} \times \text{Bobot Mata Kuliah})}{\sum \text{Bobot Mata Kuliah}}
$$

## database.py 

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash
from flask_login import UserMixin
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
        
with app.app_context():
    db.create_all()    
```
## run.py
```python
from app import app

if __name__ == '__main__':
    app.run(debug=True)

```
## __init__.py
```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager

app = Flask(__name__)
app.config['SECRET_KEY'] = 'mysecret'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///site.db'
db = SQLAlchemy(app)
login_manager = LoginManager(app)
login_manager.login_view = 'login'

from app import routes

```
## models.py
```python
from app import db
from werkzeug.security import generate_password_hash, check_password_hash
from flask_login import UserMixin

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
```
## routes.py
```python
from flask import render_template, url_for, flash, redirect, request
from app import app, db, login_manager
from app.models import User, Message
from flask_login import login_user, current_user, logout_user, login_required

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

@app.route("/")

def main():
    return render_template('main.html')

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

```

## base.html
```html
<!DOCTYPE html>
<html lang="en"> 
<head>
    <title>DevBlog - Bootstrap 5 Blog Template For Developers</title>
    
    <!-- Meta -->
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="Blog Template">
    <meta name="author" content="Xiaoying Riley at 3rd Wave Media">    
    <link rel="shortcut icon" href="favicon.ico"> 
    
    <!-- FontAwesome JS-->
	<script defer src="{{ url_for('static', filename='assets/fontawesome/js/all.min.js') }}"></script>
    
    <!-- Theme CSS -->  
    <link id="theme-style" rel="stylesheet" href="{{ url_for('static', filename='assets/css/theme-3.css') }}">

</head> 

<body data-bs-theme="dark">
    
    <header class="header text-center">	    
	    <h1 class="blog-name pt-lg-4 mb-0"><a class="no-text-decoration" href="index.html">Anthony's Blog</a></h1>
        
	    <nav class="navbar navbar-expand-lg navbar-dark" >
           
			<button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navigation" aria-controls="navigation" aria-expanded="false" aria-label="Toggle navigation">
				<span class="navbar-toggler-icon"></span>
			</button>

			<div id="navigation" class="collapse navbar-collapse flex-column" >
				<div class="profile-section pt-3 pt-lg-0">
				    <img class="profile-image mb-3 rounded-circle mx-auto" src="{{ url_for('static', filename='assets/images/profile.png')}}" alt="image" >			
					
					<div class="bio mb-3">Hi, my name is Anthony Doe. Briefly introduce yourself here. You can also provide a link to the about page.<br><a href="about.html">Find out more about me</a></div><!--//bio-->
					<ul class="social-list list-inline py-3 mx-auto">
			            <li class="list-inline-item"><a href="#"><i class="fab fa-twitter fa-fw"></i></a></li>
			            <li class="list-inline-item"><a href="#"><i class="fab fa-linkedin-in fa-fw"></i></a></li>
			            <li class="list-inline-item"><a href="#"><i class="fab fa-github-alt fa-fw"></i></a></li>
			            <li class="list-inline-item"><a href="#"><i class="fab fa-stack-overflow fa-fw"></i></a></li>
			            <li class="list-inline-item"><a href="#"><i class="fab fa-codepen fa-fw"></i></a></li>
			        </ul><!--//social-list-->
			        <hr> 
				</div><!--//profile-section-->
				
				<ul class="navbar-nav flex-column text-start">
					<li class="nav-item">
					    <a class="nav-link active" href="#"><i class="fas fa-home fa-fw me-2"></i>Blog Home <span class="sr-only">(current)</span></a>
					</li>
					<li class="nav-item">
					    <a class="nav-link" href="#"><i class="fas fa-bookmark fa-fw me-2"></i>Blog Post</a>
					</li>
					<li class="nav-item">
					    <a class="nav-link" href="#"><i class="fas fa-user fa-fw me-2"></i>About Me</a>
					</li>
				</ul>
				
				<div class="my-2 my-md-3">
				    <a class="btn btn-primary" href="#" target="_blank">Get in Touch</a>
				</div>
			</div>
		</nav>
    </header>
    
    <div class="main-wrapper">
	  {% block content %}{% endblock content %}
	    
	    <footer class="footer text-center py-2 theme-bg-dark">
		   
	        <!--/* This template is free as long as you keep the footer attribution link. If you'd like to use the template without the attribution link, you can buy the commercial license via our website: themes.3rdwavemedia.com Thank you for your support. :) */-->
            <small class="copyright">Designed with <span class="sr-only">love</span><i class="fas fa-heart" style="color: #fb866a;"></i> by <a href="https://themes.3rdwavemedia.com" target="_blank">Xiaoying Riley</a> for developers</small>
		   
	    </footer>
    
    </div><!--//main-wrapper-->

       
    <!-- Javascript -->          
    <script src="{{ url_for('static', filename='assets/plugins/popper.min.js') }}"></script> 
    <script src="{{ url_for('static', filename='assets/plugins/bootstrap/js/bootstrap.min.js') }}"></script> 

    

</body>
</html> 


```

## basedashboard.html
```html
<!doctype html>
<html lang="en">

<head>
    <title>{% block title %}ayama{% endblock title %}</title>
  <!-- Required meta tags -->
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
  <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
  <!-- Bootstrap CSS v5.2.1 -->
 
  <link href="{{ url_for('static', filename='css/cosmo.min.css') }}" rel="stylesheet"/>
  
  <script src="https://cdnjs.cloudflare.com/ajax/libs/typed.js/2.0.11/typed.min.js"></script>  
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.2/css/all.min.css"
</head>

<body>
  <header>
   
        <nav class="navbar navbar-expand-lg px-3 pe-lg-5 half-trans" id="main_menu">
            <a href="/home">
                <img class="img-fluid" src="{{ url_for('static', filename='img/logo.png') }}" alt="Logo Website"/>
    
            
            <div class="collapse navbar-collapse flex-grow-0 ms-auto" id="navbar_collapse_01">
                <ul class="navbar-nav me-auto">
                    <li class="nav-item">
                        <a class="nav-link link-primary" href="#" target="_blank">
                            <strong>Beranda</strong>
                        </a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link link-primary" href="#">
                            <strong>Game</strong>
                        </a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link link-primary" href="#">
                            <strong>Komunitas</strong>
                        </a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link link-primary" href="#">
                            <strong>Berita</strong>
                        </a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link link-primary" href="#">
                            <strong>Kontak</strong>
                        </a>
                    </li>
                </ul>
                
            </div>
        </nav><!-- place navbar here -->
  </header>
  <main>
    
    <div id="container">
      
     {% block content %}{% endblock content %}  
    </div>
</main>
{%include "sidebar.html"%}
  <footer>
    <!-- place footer here -->
  </footer>
  <!-- Bootstrap JavaScript Libraries -->
  <script src="https://cdn.jsdelivr.net/npm/@popperjs/core@2.11.6/dist/umd/popper.min.js"
    integrity="sha384-oBqDVmMz9ATKxIep9tiCxS/Z9fNfEXiDAYTujMAeBAsjFuCZSmKbSSUnQlmh/jp3" crossorigin="anonymous">
  </script>

  <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.2.1/dist/js/bootstrap.min.js"
    integrity="sha384-7VPbUDkoPSGFnVtYi0QogXtr74QeVeeIs99Qfg5YCF+TidwNdjvaKZX19NZ/e6oz" crossorigin="anonymous">
  </script>
  <script>
    var messages = [
        {% for message in messages %}
            "{{ message.message }}",
        {% endfor %}
    ];

    var typed = new Typed('.typed-text', {
        strings: messages,
        typeSpeed: 80,
        backSpeed: 50,
        loop: true
    });
</script>
</body>

</html>
```
## dashboard.html
```html
{% extends "basedashboard.html" %}

{% block title %}Typed{% endblock %}

{% block content %}
        <div class="px-5 pt-5 pb-4">
            <h3 class="text-primary text-center fw-700">Motto Kami</h3>
            <h4 class="text-info text-center mb-0"><em>"<span class="typed-text"></span>"</em></h4>
        </div>
        <div class="container mt-5">
            
            <h1 class="mb-5"></h1>

            <!-- Form untuk menambahkan pesan baru -->
            <form action="/create" method="post" class="mb-5">
                <div class="input-group">
                    <input type="text" name="new_message" class="form-control" placeholder="Enter new message">
                    <button class="btn btn-primary" type="submit">Add Message</button>
                </div>
            </form>

            <!-- Daftar pesan dan tombol untuk menghapus -->
            <ul class="list-group">
                {% for message in messages %}
                    <li class="list-group-item d-flex justify-content-between align-items-center">
                        {{ message.message }}
                        <form action="/delete/{{ message.id }}" method="post" style="display: inline;">
                            <button class="btn btn-danger" type="submit">Delete</button>
                        </form>
                    </li>
                {% endfor %}
            
    {% endblock content %}


```
## home.html
```html
{% with messages = get_flashed_messages(with_categories=true) %}
            {% if messages %}
                <div class="flash-messages">
                    {% for category, message in messages %}
                     <div class="alert alert-{{ category }}">{{ message }}</div>
                    {% endfor %} 
                </div>    
            {% endif %}
            {% endwith %}
<h1>Welcome to the Home Page!</h1>
<a href="{{ url_for('logout') }}">Logout</a>

```
## login.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login</title>

    <!-- Bootstrap CSS via CDN -->
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css">

    <!-- Optional: You can also include Bootstrap Icons via CDN -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/typed.js/2.0.11/typed.min.js"></script>  
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.18.0/font/bootstrap-icons.css">
</head>
<body>

<div class="container mt-5">
    <!-- Section: Design Block -->
    <section class="text-center text-lg-start">
      <style>
        /* Efek bayangan dan 3D pada judul */
        h2.fw-bold {
          text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.2);
          
          border-radius: 5px;
          padding: 10px 20px;
          
        }

        .card {
          animation: fadeIn 1s ease-in-out; /* Menggunakan animasi fadeIn pada elemen card */
        }
      
        body {
          background-color: gray;
        }
      
        .cascading-right {
          margin-right: -50px;
        }
      
        @media (max-width: 991.98px) {
          .cascading-right {
            margin-right: 0;
          }
        }
      
        /* Tombol Login */
        .btn-login {
          background-color: #007bff;
          background-image: linear-gradient(to bottom, #007bff, #0056b3);
          border: none;
          color: #fff;
          transition: background-color 0.3s ease;
        }
      
        .btn-login:hover {
          background-color: #0056b3;
        }
      
        /* Tombol Register */
        .btn-register {
          background-color: #28a745;
          background-image: linear-gradient(to bottom, #28a745, #1e7e34);
          border: none;
          color: #fff;
          transition: background-color 0.3s ease;
        }
      
        .btn-register:hover {
          background-color: #1e7e34;
        }
      </style>
      

        <!-- Jumbotron -->
        <div class="container py-4">
            <div class="row g-0 align-items-center">
                <div class="col-lg-6 mb-5 mb-lg-0">
                    <div class="card cascading-right" style="
                        background: hsla(0, 0%, 100%, 0.55);
                        backdrop-filter: blur(30px);
                    ">
                        <div class="card-body p-5 shadow-5 text-center">
                            {% with messages = get_flashed_messages(with_categories=true) %}
                            {% if messages %}
                            <div class="flash-messages">
                                {% for category, message in messages %}
                                <div class="alert alert-{{ category }}">{{ message }}</div>
                                {% endfor %}
                            </div>
                            {% endif %}
                            {% endwith %}
                            <h2 class="fw-bold mb-5"><span class="typed-text"></span></h2>
                            <form method="post">
                                <!-- Username input -->
                                <div class="form-outline mb-4">
                                    <input type="text" class="form-control" id="username" name="username"
                                        placeholder="Username" required>
                                </div>

                                <!-- Password input -->
                                <div class="form-outline mb-4">
                                    <input type="password" class="form-control" id="password" name="password"
                                        placeholder="Password" required>
                                </div>
                               
                                <button type="submit" class="btn btn-primary">Login</button>
                                <button type="button" class="btn  btn-register" onclick="window.location.href = '{{ url_for('register') }}'">Register</button>
                                
                            </form>
                        </div>
                    </div>
                </div>

                <div class="col-lg-6 mb-5 mb-lg-0">
                    <img src="{{url_for('static', filename='img/004.png')}}"
                        class="w-100 rounded-4 shadow-4" alt="" />
                </div>
            </div>
        </div>
        <!-- Jumbotron -->
    </section>
    <!-- Section: Design Block -->

    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</div>
<script>
  var messages = [
      {% for message in messages %}
          "{{ message.message }}",
      {% endfor %}
  ];

  var typed = new Typed('.typed-text', {
      strings: messages,
      typeSpeed: 250,
      backSpeed: 50,
      loop: true
  });
</script>
</body>
</html>

```
## main.html
```html
{% extends "base.html" %}
 {% block content %} 
	    <section class="cta-section theme-bg-light py-5">
		    <div class="container text-center single-col-max-width">
			    <h2 class="heading">DevBlog - A Blog Template Made For Developers</h2>
			    <div class="intro">Welcome to my blog. Subscribe and get my latest blog post in your inbox.</div>
			    <div class="single-form-max-width pt-3 mx-auto">
				    <form class="signup-form row g-2 g-lg-2 align-items-center">
	                    <div class="col-12 col-md-9">
	                        <label class="sr-only" for="semail">Your email</label>
	                        <input type="email" id="semail" name="semail1" class="form-control me-md-1 semail" placeholder="Enter email">
	                    </div>
	                    <div class="col-12 col-md-2">
	                        <button type="submit" class="btn btn-primary">Subscribe</button>
	                    </div>
	                </form><!--//signup-form-->
			    </div><!--//single-form-max-width-->
		    </div><!--//container-->
	    </section>
	    
	    
	    <section class="blog-list px-3 py-5 p-md-5">
		    <div class="container single-col-max-width">
			    <div class="item mb-5">
				    <div class="row g-3 g-xl-0">
					    <div class="col-2 col-xl-3">
					        <img class="img-fluid post-thumb " src="{{ url_for('static', filename='assets/images/blog/blog-post-thumb-1.jpg') }}" alt="image">
					    </div>
					    <div class="col">
						    <h3 class="title mb-1"><a class="text-link" href="#">Top 3 JavaScript Frameworks</a></h3>
						    <div class="meta mb-1"><span class="date">Published 2 days ago</span><span class="time">5 min read</span><span class="comment"><a class="text-link" href="#">8 comments</a></span></div>
						    <div class="intro">Lorem ipsum dolor sit amet, consectetuer adipiscing elit. Aenean commodo ligula eget dolor. Aenean massa. Cum sociis natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. Donec quam felis, ultricies...</div>
						    <a class="text-link" href="blog-post.html">Read more &rarr;</a>
					    </div><!--//col-->
				    </div><!--//row-->
			    </div><!--//item-->
			  
			    
			    <nav class="blog-nav nav nav-justified my-5">
				  <a class="nav-link-prev nav-item nav-link d-none rounded-left" href="#">Previous<i class="arrow-prev fas fa-long-arrow-alt-left"></i></a>
				  <a class="nav-link-next nav-item nav-link rounded" href="#">Next<i class="arrow-next fas fa-long-arrow-alt-right"></i></a>
				</nav>
				
		    </div>
	    </section>
 {% endblock content %}         
```
## register.html
```html

<form method="POST">
    <input type="text" name="username" placeholder="Username" required>
    <input type="password" name="password" placeholder="Password" required>
    <button type="submit">Register</button>
</form>

```
## sidebar.html
```html

{%block sidebar%}
<div id="sidebar">
    <a class="menu-item" href="#">
        <i class="fa-solid fa-home"></i>
        <div>HOME</div>
    </a>
    <a class="menu-item" href="#">
        <i class="fa-solid fa-newspaper"></i>
        <div>NEWS</div>
    </a>
    <a class="menu-item" href="/unit">
        <i class="fa-solid fa-school"></i>
        <div>UNITS</div>
    </a>
    <a class="menu-item" href="{{ url_for('logout')}}">
        <i class="fa-solid fa-sign-out-alt"></i>
        <div>LOGOFF</div>
    </a>
    <a class="menu-item" href="https://wa.me/6281522737386" target="_blank">
        <i class="fa-solid fa-message"></i>
        <div>CHAT</div>
    </a>
</div>
{%endblock sidebar%}

```
