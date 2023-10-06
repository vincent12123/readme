# Aplikasi Nilai Akademik
## Struktur app
```
nakademik
├── app.py
├── database.py
├── instance
│   └── grades.db
├── static
│   └── style.css
└── templates
    ├── base.html
    ├── delete.html
    ├── index.html
    └── laporan.html

```

## buat database.py 
```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

# create the extension
db = SQLAlchemy()
# create the app
app = Flask(__name__)
# configure the SQLite database, relative to the app instance folder
app.config["SQLALCHEMY_DATABASE_URI"] = "sqlite:///grades.db"
# initialize the app with the extension
db.init_app(app)

class Grade(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    subject = db.Column(db.String(50), nullable=False)
    score = db.Column(db.Float, nullable=False)
    semester = db.Column(db.String(20), nullable=False)

with app.app_context():
    db.create_all()    
```

## buat app.py
```python

from flask import Flask, render_template, request, redirect, url_for, flash, send_from_directory
from flask_sqlalchemy import SQLAlchemy
import pandas as pd
import os

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///grades.db'
app.config['SECRET_KEY'] = 'some_secret_key'
db = SQLAlchemy(app)

class Grade(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    subject = db.Column(db.String(50), nullable=False)
    score = db.Column(db.Float, nullable=False)
    semester = db.Column(db.String(20), nullable=False)

@app.route('/')
def index():
    grades = Grade.query.all()
    average = sum([grade.score for grade in grades]) / len(grades) if grades else 0
    feedback = "Bagus!" if average >= 85 else "Tetap semangat dan tingkatkan lagi!"
    return render_template('index.html', grades=grades, average=average, feedback=feedback)

@app.route('/add_grade', methods=['POST'])
def add_grade():
    subject = request.form.get('subject')
    score = float(request.form.get('score'))
    semester = request.form.get('semester')
    
    if 0 <= score <= 100:
        grade = Grade(subject=subject, score=score, semester=semester)
        db.session.add(grade)
        db.session.commit()
        flash('Nilai berhasil ditambahkan!', 'success')
    else:
        flash('Nilai tidak valid! Harus antara 0-100.', 'danger')
    return redirect(url_for('index'))

@app.route('/edit_grade/<int:id>', methods=['POST'])
def edit_grade(id):
    grade = Grade.query.get(id)
    grade.subject = request.form.get('subject')
    grade.score = float(request.form.get('score'))
    grade.semester = request.form.get('semester')
    db.session.commit()
    flash('Nilai berhasil diperbarui!', 'success')
    return redirect(url_for('index'))

@app.route('/delete_grade/<int:id>')
def delete_grade(id):
    grade = Grade.query.get(id)
    db.session.delete(grade)
    db.session.commit()
    flash('Nilai berhasil dihapus!', 'success')
    return redirect(url_for('index'))

@app.route('/export_data')
def export_data():
    grades = Grade.query.all()
    df = pd.DataFrame([(grade.subject, grade.score, grade.semester) for grade in grades], columns=["Subject", "Score", "Semester"])
    filename = "grades_export.csv"
    df.to_csv(filename, index=False)
    return send_from_directory(os.getcwd(), filename, as_attachment=True)
@app.route('/delete_page')
def delete_page():
    grades = Grade.query.all()
    return render_template('delete.html', grades=grades)

@app.route('/laporan')
def laporan():
    grades = Grade.query.all()
    average = sum([grade.score for grade in grades]) / len(grades) if grades else 0
    total_subjects = len(grades)
    return render_template('laporan.html', grades=grades, average=average, total_subjects=total_subjects)

@app.route('/export_excel')
def export_excel():
    grades = Grade.query.all()
    df = pd.DataFrame([(grade.subject, grade.score, grade.semester) for grade in grades], columns=["Subject", "Score", "Semester"])
    filename = "grades_export.xlsx"
    df.to_excel(filename, index=False)
    return send_from_directory(os.getcwd(), filename, as_attachment=True)

if __name__ == "__main__":
    
    app.run(debug=True)

```

## base.html
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Kalkulator Nilai Akademik</title>
    <link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">

    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-light bg-light">
        <div class="container">
            <a class="navbar-brand" href="{{ url_for('index') }}">Kalkulator Nilai Akademik</a>
            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav" aria-controls="navbarNav" aria-expanded="false" aria-label="Toggle navigation">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav">
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('index') }}">Beranda</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('laporan') }}">Laporan</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="{{ url_for('delete_page') }}">Hapus Nilai</a>
                    </li>
                </ul>
            </div>
        </div>
    </nav>
    
<div class="container mt-5">
    {% block content %}{% endblock %}
</div>

<script src="https://cdn.jsdelivr.net/npm/@popperjs/core@2.10.1/dist/umd/popper.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"></script>

</body>
</html>

```
## delete.html
```html
{% extends "base.html" %}

{% block content %}
<h1 class="mb-4">Hapus Nilai</h1>

<!-- Tabel Nilai -->
<table class="table table-bordered">
    <thead>
        <tr>
            <th>Mata Pelajaran</th>
            <th>Nilai</th>
            <th>Semester</th>
            <th>Aksi</th>
        </tr>
    </thead>
    <tbody>
        {% for grade in grades %}
        <tr>
            <td>{{ grade.subject }}</td>
            <td>{{ grade.score }}</td>
            <td>{{ grade.semester }}</td>
            <td>
                <a href="{{ url_for('delete_grade', id=grade.id) }}" class="btn btn-danger btn-sm">Hapus</a>
            </td>
        </tr>
        {% endfor %}
    </tbody>
</table>

<a href="{{ url_for('index') }}" class="btn btn-primary mt-3">Kembali ke Halaman Utama</a>
{% endblock %}

```
## index.html
```html
{% extends "base.html" %}

{% block content %}
<div class="container mt-5">
    <h1 class="mb-4">Kalkulator Nilai Akademik</h1>

    <!-- Formulir Tambah/Edit Nilai -->
    <form action="{{ url_for('add_grade') }}" method="post" class="mb-4">
        <div class="mb-3 row">
            <label for="subject" class="col-sm-2 col-form-label">Mata Pelajaran</label>
            <div class="col-sm-10">
                <input type="text" name="subject" id="subject" placeholder="Mata Pelajaran" class="form-control" required>
            </div>
        </div>

        <div class="mb-3 row">
            <label for="score" class="col-sm-2 col-form-label">Nilai</label>
            <div class="col-sm-10">
                <input type="number" name="score" id="score" placeholder="Nilai" min="0" max="100" class="form-control" required>
            </div>
        </div>

        <div class="mb-3 row">
            <label for="semester" class="col-sm-2 col-form-label">Semester</label>
            <div class="col-sm-10">
                <select name="semester" id="semester" class="form-control">
                    <option value="Semester 1">Semester 1</option>
                    <option value="Semester 2">Semester 2</option>
                </select>
            </div>
        </div>

        <div class="mb-3 row">
            <div class="col-sm-10 offset-sm-2">
                <button type="submit" class="btn btn-primary">Tambah Nilai</button>
            </div>
        </div>
    </form>

    <!-- Rata-rata dan Feedback -->
    <div class="alert alert-info" role="alert">
        <strong>Rata-rata Nilai:</strong> {{ average }}
        <br>
        <strong>Feedback:</strong> {{ feedback }}
    </div>

    <!-- Grafik Nilai dengan Chart.js -->
    <div class="card mb-4">
        <div class="card-body">
            <canvas id="gradesChart" width="400" height="200"></canvas>
        </div>
    </div>

    <!-- Tombol ke Halaman Hapus Nilai dan Ekspor Data -->
    <a href="{{ url_for('delete_page') }}" class="btn btn-danger">Pergi ke Halaman Hapus Nilai</a>
    <a href="{{ url_for('export_data') }}" class="btn btn-success">Unduh Data</a>
</div>

<script>
    var ctx = document.getElementById('gradesChart').getContext('2d');
    var gradesChart = new Chart(ctx, {
        type: 'bar',
        data: {
            labels: {{ grades|map(attribute='subject')|list|tojson }},
            datasets: [{
                label: 'Nilai',
                data: {{ grades|map(attribute='score')|list|tojson }},
                backgroundColor: 'rgba(75, 192, 192, 0.2)',
                borderColor: 'rgba(75, 192, 192, 1)',
                borderWidth: 1
            }]
        },
        options: {
            scales: {
                y: {
                    beginAtZero: true,
                    max: 100
                }
            }
        }
    });
</script>
{% endblock %}

```
## laporan.html
```html
{% extends "base.html" %}

{% block content %}
<div class="container mt-5">
    <h1 class="mb-4">Laporan Nilai Akademik</h1>

    <!-- Ringkasan Laporan -->
    <div class="row mb-4">
        <div class="col-md-6">
            <div class="card">
                <div class="card-body">
                    <h5 class="card-title">Rata-rata Nilai</h5>
                    <p class="card-text">{{ average }}</p>
                </div>
            </div>
        </div>
        <div class="col-md-6">
            <div class="card">
                <div class="card-body">
                    <h5 class="card-title">Jumlah Mata Pelajaran</h5>
                    <p class="card-text">{{ total_subjects }}</p>
                </div>
            </div>
        </div>
    </div>

    <!-- Grafik Nilai dengan Chart.js -->
    <div class="card mb-4">
        <div class="card-body">
            <h5 class="card-title">Visualisasi Nilai</h5>
            <canvas id="gradesChart" width="400" height="200"></canvas>
        </div>
    </div>
    <a href="{{ url_for('export_excel') }}" class="btn btn-info mt-3">Unduh dalam Format Excel</a>
    <a href="{{ url_for('index') }}" class="btn btn-primary">Kembali ke Halaman Utama</a>
</div>

<script>
    var ctx = document.getElementById('gradesChart').getContext('2d');
    var gradesChart = new Chart(ctx, {
        type: 'bar',
        data: {
            labels: {{ grades|map(attribute='subject')|list|tojson }},
            datasets: [{
                label: 'Nilai',
                data: {{ grades|map(attribute='score')|list|tojson }},
                backgroundColor: 'rgba(75, 192, 192, 0.2)',
                borderColor: 'rgba(75, 192, 192, 1)',
                borderWidth: 1
            }]
        },
        options: {
            scales: {
                y: {
                    beginAtZero: true,
                    max: 100
                }
            }
        }
    });
</script>
{% endblock %}

```
## style.css
```css
body {
    font-family: 'Arial', sans-serif;
    background-color: #f4f4f4;
    margin: 0;
    padding: 0;
}

.container {
    background-color: white;
    border-radius: 5px;
    box-shadow: 0px 0px 10px rgba(0, 0, 0, 0.1);
    padding: 20px;
    margin-top: 50px;
}

h1 {
    border-bottom: 2px solid #ddd;
    padding-bottom: 10px;
    margin-bottom: 20px;
}

table {
    margin-top: 20px;
}

canvas#gradesChart {
    margin-top: 40px;
    margin-bottom: 20px;
}

```
