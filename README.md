# baymax.bt-backend
import os
from flask import Flask, render_template, request, jsonify, session, url_for
from werkzeug.security import generate_password_hash, check_password_hash
from flask_sqlalchemy import SQLAlchemy
import bcrypt
from models import Users
from flask import Flask
from models import db, Users, init_app
from models import Symptoms, Illnesses
from sqlalchemy import join
import analysis
import models
from flask import request

app = Flask(__name__, static_folder='../baymax-healt-assistant front end/static', template_folder='../baymax-healt'
                                                                                                  '-assistant front '
                                                                                                  'end/templates')
app.config['SQLALCHEMY_DATABASE_URI'] = "mysql://root:jj1995123@localhost:3306/baymax_db"
app.config['TEMPLATE_FOLDER'] = os.path.join(os.path.dirname(__file__), 'templates')


@app.route('/')
def index():
    static_url = url_for('static', filename='website.css')
    print(f"Generated static URL for website.css: {static_url}")
    return render_template("website.html")

db = SQLAlchemy(app)


@app.route('/register', methods=['POST'])
def register():
    email = request.form['email']
    password = request.form['password']

    try:
        password_hash = generate_password_hash(password)
        new_user = Users(email=email, password_hash=password_hash)
        db.session.add(new_user)
        db.session.commit()
        return "Registration successful!"
    except Exception as e:
        print(f"Error during registration: {e}")
        return "Registration failed. Please try again.", 500


@app.route('/login', methods=['POST'])
def login():
    email = request.form['email']
    password = request.form['password']

    users = Users.query.filter_by(email=email).first()
    if user and check_password_hash(user.password_hash, password):
        session['user_id'] = user.id
        return jsonify({"message": "Login successful!"})
    else:
        return jsonify({"error": "Invalid email or password"}), 401


@app.route("/submit_symptoms", methods=["POST"])
def submit_symptoms():
    if 'user_id' not in session:
        return jsonify({"error": "Unauthorized access"}), 401

    symptoms = request.form["symptoms"]
    selected_symptoms = request.form.getlist("symptoms")
    illness_data = analyze_symptoms(selected_symptoms)

    if isinstance(illness_data, str):
        return jsonify({"error": illness_data}), 500
    else:
        return illness_analysis


analyze_symptoms = {
    "Malaria": {
        "symptoms": ["headache", "fever", "chills", "sweating"],
        "description": "A mosquito-borne infectious disease.",
        "related_illnesses": ["Dengue fever", "Typhoid fever"]
    },
    "Common cold": {
        "symptoms": ["cough", "sore throat", "runny nose", "congestion"],
        "description": "A viral infection of the upper respiratory tract.",
        "related_illnesses": ["Influenza (flu)", "Sinusitis"]
    },
    "Stroke": {
        "symptoms": ["sudden numbness", "confusion", "trouble speaking", "trouble walking"],
        "description": "A sudden interruption of blood supply to part of the brain.",
        "related_illnesses": ["Transient ischemic attack (TIA)"]
    },
    "Eczema": {
        "symptoms": ["itching", "rash", "dry skin"],
        "description": "A chronic inflammatory skin condition.",
        "related_illnesses": ["Psoriasis", "Hives"]
    },
    "Heart Disease": {
        "symptoms": ["chest pain", "shortness of breath", "fatigue"],
        "description": "A group of conditions affecting the heart and blood vessels.",
        "related_illnesses": ["Angina", "Heart failure"]
    }
}


def analyze_symptoms(symptoms):
    session = db.session

    illnesses = (
        session.query(Illness)
        .join(Symptoms, Illnesses.symptoms)
        .filter(Symptoms.name.in_(symptoms))
        .all()
    )

    illness_data = []
    for illness in illnesses:
        illness_data.append({
            "name": illness.name,
        })
    return jsonify(illnesses=illness_data)


if __name__ == '__main__':
    app.run(debug=True)
