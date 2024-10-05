# cloud-computing-assignment-2
# code:
from flask import Flask, request, redirect, url_for, render_template_string, session, flash
import sqlite3

app = Flask(__name__)

# SQLite database file path
DATABASE = '/var/www/html/flaskapp/users.db'

# Function to create the users table if it doesn't exist
def init_db():
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL,
            password TEXT NOT NULL,
            firstname TEXT NOT NULL,
            lastname TEXT NOT NULL
        )
    ''')
    conn.commit()
    conn.close()

# Initialize the database
init_db()

# Route to display the sign-up and sign-in forms
@app.route('/')
def index():
    return render_template_string('''
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Sign Up / Sign In</title>
    </head>
    <body>
        <h1>Sign Up</h1>
        <form method="post" action="/register">
            <label for="username">Username:</label><br>
            <input type="text" id="username" name="username" required><br><br>
            
            <label for="email">Email:</label><br>
            <input type="email" id="email" name="email" required><br><br>
            
            <label for="password">Password:</label><br>
            <input type="password" id="password" name="password" required><br><br>
            
            <label for="firstname">First Name:</label><br>
            <input type="text" id="firstname" name="firstname" required><br><br>
            
            <label for="lastname">Last Name:</label><br>
            <input type="text" id="lastname" name="lastname" required><br><br>
            
            <input type="submit" value="Sign Up">
        </form>

        <h1>Sign In</h1>
        <form method="post" action="/signin">
            <label for="username">Username:</label><br>
            <input type="text" id="username" name="username" required><br><br>
            
            <label for="password">Password:</label><br>
            <input type="password" id="password" name="password" required><br><br>
            
            <input type="submit" value="Sign In">
        </form>

        <!-- Display error and success messages -->
        {% with messages = get_flashed_messages(with_categories=true) %}
          {% if messages %}
            <ul>
              {% for category, message in messages %}
                <li style="color: {% if category == 'error' %}red{% else %}green{% endif %};">{{ message }}</li>
              {% endfor %}
            </ul>
          {% endif %}
        {% endwith %}
    </body>
    </html>
    ''')

# Route to handle user registration
@app.route('/register', methods=['POST'])
def register():
    username = request.form['username']
    email = request.form['email']
    password = request.form['password']
    firstname = request.form['firstname']
    lastname = request.form['lastname']

    try:
        conn = sqlite3.connect(DATABASE)
        cursor = conn.cursor()
        cursor.execute('''
            INSERT INTO users (username, email, password, firstname, lastname)
            VALUES (?, ?, ?, ?, ?)
        ''', (username, email, password, firstname, lastname))
        conn.commit()
        conn.close()
        flash('Registration successful! Please sign in.', 'success')
        return redirect(url_for('index'))
    except sqlite3.IntegrityError:
        conn.close()
        flash('Username or email already exists. Please try a different one.', 'error')
        return redirect(url_for('index'))
    except Exception as e:
        conn.close()
        flash(f'An error occurred: {e}', 'error')
        return redirect(url_for('index'))

# Route to handle user sign-in
@app.route('/signin', methods=['POST'])
def signin():
    username = request.form['username']
    password = request.form['password']

    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM users WHERE username=? AND password=?', (username, password))
    user = cursor.fetchone()
    conn.close()

    if user:
        session['username'] = username  # Save username in session
        flash('Sign in successful!', 'success')
        return redirect(url_for('profile', username=username))
    else:
        flash('Invalid username or password. Please try again.', 'error')
        return redirect(url_for('index'))

# Route to display user profile
@app.route('/profile/<username>')
def profile(username):
    if 'username' not in session or session['username'] != username:
        flash('You need to sign in to view your profile.', 'error')
        return redirect(url_for('index'))

    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE username=?", (username,))
    user = cursor.fetchone()
    conn.close()

    if user:
        return render_template_string('''
        <!DOCTYPE html>
        <html lang="en">
        <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <title>User Profile</title>
        </head>
        <body>
            <h1>User Profile</h1>
            <p><strong>Username:</strong> {{ user[1] }}</p>
            <p><strong>First Name:</strong> {{ user[4] }}</p>
            <p><strong>Last Name:</strong> {{ user[5] }}</p>
            <p><strong>Email:</strong> {{ user[2] }}</p>
            <a href="/">Back to Home</a>
        </body>
        </html>
        ''', user=user)
    else:
        flash('User not found.', 'error')
        return redirect(url_for('index'))

if __name__ == '__main__':
    app.run(debug=True)

