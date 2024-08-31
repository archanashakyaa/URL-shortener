To create a URL shortener with user authentication and click tracking, we'll use the following technologies:

1. *Flask*: For the web framework.
2. *Flask-SQLAlchemy*: For database interactions.
3. *Flask-Login*: For user authentication.
4. *Flask-Migrate*: For handling database migrations.
5. *SQLite*: For a simple, lightweight database.

### Step 1: Install Required Libraries

Make sure to install the required libraries by running:

bash
pip install Flask Flask-SQLAlchemy Flask-Login Flask-Migrate


### Step 2: Create the Flask Application

1. *Set up the project structure:*


url_shortener/
│
├── app.py
├── models.py
├── forms.py
├── templates/
│   ├── index.html
│   ├── login.html
│   ├── register.html
│   └── dashboard.html
└── migrations/


2. **Create the app.py file:**

python
from flask import Flask, render_template, redirect, url_for, request, flash
from flask_sqlalchemy import SQLAlchemy
from flask_login import LoginManager, UserMixin, login_user, login_required, logout_user, current_user
from flask_migrate import Migrate
from werkzeug.security import generate_password_hash, check_password_hash
import string
import random

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your_secret_key'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///db.sqlite'

db = SQLAlchemy(app)
migrate = Migrate(app, db)
login_manager = LoginManager()
login_manager.init_app(app)
login_manager.login_view = 'login'

# Models
class User(UserMixin, db.Model):
    id = db.Column(db.Integer, primary_key=True)
    email = db.Column(db.String(150), unique=True, nullable=False)
    password = db.Column(db.String(150), nullable=False)
    urls = db.relationship('URL', backref='owner', lazy=True)

class URL(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    original_url = db.Column(db.String(500), nullable=False)
    short_url = db.Column(db.String(10), unique=True, nullable=False)
    clicks = db.Column(db.Integer, default=0)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)

@login_manager.user_loader
def load_user(user_id):
    return User.query.get(int(user_id))

def generate_short_url():
    characters = string.ascii_letters + string.digits
    short_url = ''.join(random.choice(characters) for _ in range(6))
    if URL.query.filter_by(short_url=short_url).first():
        return generate_short_url()
    return short_url

# Routes
@app.route('/')
def index():
    return render_template('index.html')

@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        email = request.form.get('email')
        password = request.form.get('password')
        user = User.query.filter_by(email=email).first()

        if user:
            flash('Email address already exists')
            return redirect(url_for('register'))

        new_user = User(email=email, password=generate_password_hash(password, method='sha256'))
        db.session.add(new_user)
        db.session.commit()

        flash('Account created successfully')
        return redirect(url_for('login'))

    return render_template('register.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        email = request.form.get('email')
        password = request.form.get('password')
        user = User.query.filter_by(email=email).first()

        if user and check_password_hash(user.password, password):
            login_user(user)
            return redirect(url_for('dashboard'))

        flash('Invalid email or password')
        return redirect(url_for('login'))

    return render_template('login.html')

@app.route('/dashboard', methods=['GET', 'POST'])
@login_required
def dashboard():
    if request.method == 'POST':
        original_url = request.form.get('original_url')
        short_url = generate_short_url()
        new_url = URL(original_url=original_url, short_url=short_url, owner=current_user)
        db.session.add(new_url)
        db.session.commit()
        flash('Short URL created successfully')
        return redirect(url_for('dashboard'))

    urls = URL.query.filter_by(user_id=current_user.id).all()
    return render_template('dashboard.html', urls=urls)

@app.route('/<short_url>')
def redirect_to_url(short_url):
    url = URL.query.filter_by(short_url=short_url).first_or_404()
    url.clicks += 1
    db.session.commit()
    return redirect(url.original_url)

@app.route('/logout')
@login_required
def logout():
    logout_user()
    return redirect(url_for('index'))

if __name__ == '__main__':
    app.run(debug=True)


3. **Create the models.py file:**

The models for User and URL are already defined in app.py. To make the code modular, you can move these models to a separate models.py file and import them in app.py.

4. **Create templates for HTML files:**

- **index.html:**
    html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>URL Shortener</title>
    </head>
    <body>
        <h1>Welcome to URL Shortener</h1>
        <a href="{{ url_for('login') }}">Login</a> | 
        <a href="{{ url_for('register') }}">Register</a>
    </body>
    </html>
    

- **register.html:**
    html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Register</title>
    </head>
    <body>
        <h1>Register</h1>
        <form method="POST">
            <input type="email" name="email" placeholder="Email" required>
            <input type="password" name="password" placeholder="Password" required>
            <button type="submit">Register</button>
        </form>
    </body>
    </html>
    

- **login.html:**
    html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Login</title>
    </head>
    <body>
        <h1>Login</h1>
        <form method="POST">
            <input type="email" name="email" placeholder="Email" required>
            <input type="password" name="password" placeholder="Password" required>
            <button type="submit">Login</button>
        </form>
    </body>
    </html>
    

- **dashboard.html:**
    html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Dashboard</title>
    </head>
    <body>
        <h1>Dashboard</h1>
        <form method="POST">
            <input type="url" name="original_url" placeholder="Enter URL to shorten" required>
            <button type="submit">Shorten</button>
        </form>
        <h2>Your URLs</h2>
        <ul>
        {% for url in urls %}
            <li><a href="{{ url.short_url }}">{{ url.short_url }}</a> - {{ url.original_url }} ({{ url.clicks }} clicks)</li>
        {% endfor %}
        </ul>
        <a href="{{ url_for('logout') }}">Logout</a>
    </body>
    </html>
    

### Step 3: Run the Flask Application

1. *Initialize the Database:*

bash
flask db init
flask db migrate -m "Initial migration."
flask db upgrade


2. *Run the Application:*

bash
python app.py


Visit http://127.0.0.1:5000/ in your browser to see the application.

### Additional Enhancements

- *Analytics*: Add more detailed analytics to track clicks by date, time, and location.
- *Improved UI*: Use frontend frameworks like Bootstrap or React for a better user experience.
- *User Management*: Add functionality for users to manage their URLs, delete them, or see detailed statistics.
