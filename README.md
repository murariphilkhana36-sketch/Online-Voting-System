from flask import Flask, render_template, request, redirect, session
import sqlite3

app = Flask(__name__)
app.secret_key = "secret123"

# Initialize DB
def init_db():
    conn = sqlite3.connect('database.db')
    c = conn.cursor()

    c.execute('''CREATE TABLE IF NOT EXISTS users(
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    username TEXT,
                    password TEXT,
                    voted INTEGER DEFAULT 0)''')

    c.execute('''CREATE TABLE IF NOT EXISTS votes(
                    candidate TEXT,
                    count INTEGER)''')

    # Insert candidates if not exists
    candidates = ["Alice", "Bob", "Charlie"]
    for candi in candidates:
        c.execute("INSERT OR IGNORE INTO votes(candidate, count) VALUES (?, 0)", (candi,))

    conn.commit()
    conn.close()

init_db()

# Login Page
@app.route("/", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        username = request.form["username"]
        password = request.form["password"]

        conn = sqlite3.connect('database.db')
        c = conn.cursor()

        c.execute("SELECT * FROM users WHERE username=? AND password=?", (username, password))
        user = c.fetchone()

        if user:
            session["user"] = username
            return redirect("/vote")
        else:
            return "Invalid credentials"

    return render_template("login.html")

# Vote Page
@app.route("/vote", methods=["GET", "POST"])
def vote():
    if "user" not in session:
        return redirect("/")

    conn = sqlite3.connect('database.db')
    c = conn.cursor()

    c.execute("SELECT voted FROM users WHERE username=?", (session["user"],))
    voted = c.fetchone()[0]

    if voted == 1:
        return "You have already voted!"

    if request.method == "POST":
        candidate = request.form["candidate"]

        c.execute("UPDATE votes SET count = count + 1 WHERE candidate=?", (candidate,))
        c.execute("UPDATE users SET voted = 1 WHERE username=?", (session["user"],))

        conn.commit()
        conn.close()

        return redirect("/result")

    c.execute("SELECT candidate FROM votes")
    candidates = c.fetchall()

    return render_template("vote.html", candidates=candidates)

# Result Page
@app.route("/result")
def result():
    conn = sqlite3.connect('database.db')
    c = conn.cursor()

    c.execute("SELECT * FROM votes")
    results = c.fetchall()

    return render_template("result.html", results=results)

# Register (for testing)
@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "POST":
        username = request.form["username"]
        password = request.form["password"]

        conn = sqlite3.connect('database.db')
        c = conn.cursor()

        c.execute("INSERT INTO users(username, password) VALUES (?, ?)", (username, password))
        conn.commit()
        conn.close()

        return redirect("/")

    return '''
    <form method="post">
        Username: <input name="username"><br>
        Password: <input name="password"><br>
        <button type="submit">Register</button>
    </form>
    '''

if __name__ == "__main__":
    app.run(debug=True)
