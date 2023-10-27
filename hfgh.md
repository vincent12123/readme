from flask import Flask, render_template, request, redirect, url_for, flash
from flask_sqlalchemy import SQLAlchemy
from sqlalchemy import Column, Integer, String, Float, ForeignKey
from sqlalchemy.orm import relationship
from wtforms import Form, FloatField, IntegerField, SelectField, validators
from flask_wtf import FlaskForm
from wtforms.validators import DataRequired

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///your_database.db'
app.secret_key = 'your_secret_key'
db = SQLAlchemy(app)

# Model Tabel
class Siswa(db.Model):
    __tablename__ = 'siswa'
    id = Column(Integer, primary_key=True)
    nama = Column(String(50))
    rata_rata = Column(Float)
    nilai = relationship('Nilai', backref='siswa', lazy=True)

class MataPelajaran(db.Model):
    __tablename__ = 'mata_pelajaran'
    id = Column(Integer, primary_key=True)
    nama = Column(String(50))
    bobot = Column(Float)
    bobot_tugas_harian = Column(Float)
    bobot_tugas_proyek = Column(Float)
    bobot_uts = Column(Float)
    bobot_uas = Column(Float)
    nilai = relationship('Nilai', backref='mata_pelajaran', lazy=True)

class Nilai(db.Model):
    __tablename__ = 'nilai'
    id = Column(Integer, primary_key=True)
    siswa_id = Column(Integer, ForeignKey('siswa.id'), nullable=False)
    mata_pelajaran_id = Column(Integer, ForeignKey('mata_pelajaran.id'), nullable=False)
    tugas_harian = Column(Float)
    tugas_proyek = Column(Float)
    uts = Column(Float)
    uas = Column(Float)
    nilai_akhir = Column(Float)

class NilaiForm(FlaskForm):
    siswa = SelectField('Siswa', coerce=int)
    mata_pelajaran = SelectField('Mata Pelajaran', coerce=int)
    tugas_harian = FloatField('Tugas Harian', validators=[DataRequired()])
    tugas_proyek = FloatField('Tugas Proyek', validators=[DataRequired()])
    uts = FloatField('UTS', validators=[DataRequired()])
    uas = FloatField('UAS', validators=[DataRequired()])

def hitung_nilai_akhir(nilai):
    mata_pelajaran = MataPelajaran.query.get(nilai.mata_pelajaran_id)
    bobot = mata_pelajaran.bobot
    bobot_tugas_harian = mata_pelajaran.bobot_tugas_harian
    bobot_tugas_proyek = mata_pelajaran.bobot_tugas_proyek
    bobot_uts = mata_pelajaran.bobot_uts
    bobot_uas = mata_pelajaran.bobot_uas

    nilai.nilai_akhir = (
        nilai.tugas_harian * bobot_tugas_harian +
        nilai.tugas_proyek * bobot_tugas_proyek +
        nilai.uts * bobot_uts +
        nilai.uas * bobot_uas
    ) * bobot
    db.session.commit()

@app.route('/input_nilai', methods=['GET', 'POST'])
def input_nilai():
    form = NilaiForm()
    form.siswa.choices = [(siswa.id, siswa.nama) for siswa in Siswa.query.all()]
    form.mata_pelajaran.choices = [(mapel.id, mapel.nama) for mapel in MataPelajaran.query.all()]

    if form.validate_on_submit():
        nilai = Nilai(
            mata_pelajaran_id=form.mata_pelajaran.data,
            siswa_id=form.siswa.data,
            tugas_harian=form.tugas_harian.data,
            tugas_proyek=form.tugas_proyek.data,
            uts=form.uts.data,
            uas=form.uas.data
        )
        db.session.add(nilai)
        db.session.commit()
        hitung_nilai_akhir(nilai)
        flash('Nilai telah berhasil disimpan', 'success')
        return redirect(url_for('index'))
    return render_template('input_nilai.html', form=form)

@app.route('/laporan')
def laporan():
    siswa_list = Siswa.query.all()
    return render_template('laporan.html', siswa_list=siswa_list)

@app.route('/')
def index():
    return redirect(url_for('laporan'))

if __name__ == '__main__':
    db.create_all()
    app.run(debug=True)
