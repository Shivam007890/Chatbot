from flask import Flask, request, jsonify, render_template_string
from flask_cors import CORS
import sqlite3
from datetime import datetime
import json

app = Flask(__name__)
CORS(app)  # Enable CORS for all domains

# Initialize SQLite Database
def init_db():
    conn = sqlite3.connect('issues.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS issues
                 (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  name TEXT,
                  constituency TEXT,
                  contact TEXT,
                  issue_category TEXT,
                  issue_description TEXT,
                  timestamp TEXT)''')
    conn.commit()
    conn.close()

init_db()

# Test Home Route
@app.route('/')
def home():
    return """
    Flask Chatbot is Running!<br>
    <a href="/chatbot">Go to Chatbot Interface</a><br>
    <a href="/dashboard">View Dashboard</a>
    """

# Chatbot Response Logic
def chatbot_response(message, user_data):
    message = message.lower().strip()
    responses = {
        "hi": "Hello! I'm here to help with constituency issue redressal. May I have your name, please?",
        "hello": "Hi there! I'm your constituency issue redressal assistant. What's your name?",
        "bye": "Thank you for reaching out! Your issue has been recorded, and we'll look into it. Goodbye!",
    }

    # Default response if message not in predefined responses
    if message in responses:
        return responses[message]

    # Handle conversation flow based on user_data state
    if "step" not in user_data:
        user_data["step"] = "get_name"
        return responses["hi"]

    if user_data["step"] == "get_name":
        user_data["name"] = message
        user_data["step"] = "get_constituency"
        return f"Nice to meet you, {message}! Which constituency are you from?"

    if user_data["step"] == "get_constituency":
        user_data["constituency"] = message
        user_data["step"] = "get_contact"
        return f"Thank you, {user_data['name']}. Please provide your contact number so we can follow up with you."

    if user_data["step"] == "get_contact":
        user_data["contact"] = message
        user_data["step"] = "get_issue_category"
        return f"Thanks for sharing, {user_data['name']}. What issue would you like to report? Please choose a category:\n1. Infrastructure (roads, transport)\n2. Water Supply and Sanitation\n3. Electricity and Power\n4. Healthcare Access\n5. Others"

    if user_data["step"] == "get_issue_category":
        categories = {
            "1": "Infrastructure",
            "2": "Water Supply and Sanitation",
            "3": "Electricity and Power",
            "4": "Healthcare Access",
            "5": "Others"
        }
        if message in categories:
            user_data["issue_category"] = categories[message]
            user_data["step"] = "get_issue_description"
            return f"I see you're facing an issue with {categories[message]}. Can you please describe the problem in detail?"
        return "Please select a valid category by typing the number (1-5)."

    if user_data["step"] == "get_issue_description":
        user_data["issue_description"] = message
        user_data["step"] = "completed"

        # Save the issue to the database
        conn = sqlite3.connect('issues.db')
        c = conn.cursor()
        c.execute('''INSERT INTO issues (name, constituency, contact, issue_category, issue_description, timestamp)
                     VALUES (?, ?, ?, ?, ?, ?)''',
                  (user_data["name"], user_data["constituency"], user_data["contact"],
                   user_data["issue_category"], user_data["issue_description"],
                   datetime.now().strftime("%Y-%m-%d %H:%M:%S")))
        conn.commit()
        conn.close()

        # Provide a tailored response based on the issue category
        if user_data["issue_category"] == "Infrastructure":
            return f"Thank you, {user_data['name']}, for reporting the infrastructure issue in {user_data['constituency']}. We understand how frustrating poor roads or transport can be. Your issue has been recorded, and we'll escalate it to the relevant authorities. You'll hear from us soon!"
        elif user_data["issue_category"] == "Water Supply and Sanitation":
            return f"Thank you, {user_data['name']}, for reporting the water supply or sanitation issue in {user_data['constituency']}. Access to clean water and sanitation is critical, and we're sorry you're facing this problem. Your issue has been recorded, and we'll follow up soon!"
        elif user_data["issue_category"] == "Electricity and Power":
            return f"Thank you, {user_data['name']}, for reporting the electricity issue in {user_data['constituency']}. Reliable power is essential, and we’ll work to address this. Your issue has been recorded, and we'll get back to you soon!"
        elif user_data["issue_category"] == "Healthcare Access":
            return f"Thank you, {user_data['name']}, for reporting the healthcare access issue in {user_data['constituency']}. Access to healthcare is a priority, and we're sorry for the inconvenience. Your issue has been recorded, and we'll escalate it to the relevant department!"
        else:
            return f"Thank you, {user_data['name']}, for reporting your concern in {user_data['constituency']}. We’ve recorded your issue under 'Others,' and we'll look into it. You'll hear from us soon!"

    return "I'm sorry, I didn't understand that. Can you please repeat?"

# Chat Route (API Endpoint for Chatbot)
@app.route('/chat', methods=['POST'])
def chat():
    data = request.get_json()
    if not data or "message" not in data:
        return jsonify({"error": "No message provided"}), 400
    user_message = data.get("message", "").strip()
    if not user_message:
        return jsonify({"reply": "Please send a valid message."}), 400

    # Simulate user session with a simple dictionary (in a real app, use session management)
    user_data = data.get("user_data", {})
    bot_reply = chatbot_response(user_message, user_data)
    return jsonify({"reply": bot_reply, "user_data": user_data})

# Route to Serve the Chatbot Web Interface
@app.route('/chatbot')
def chatbot_page():
    return render_template_string("""
    <!DOCTYPE html>
    <html>
    <head>
        <title>Constituency Issue Redressal Chatbot</title>
        <style>
            body {
                font-family: Arial, sans-serif;
                max-width: 800px;
                margin: 0 auto;
                padding: 20px;
                background-color: #f4f4f4;
            }
            h1 {
                text-align: center;
                color: #333;
            }
            #chatbox {
                border: 1px solid #ccc;
                padding: 10px;
                height: 400px;
                overflow-y: scroll;
                background-color: white;
                margin-bottom: 20px;
                border-radius: 5px;
            }
            .message {
                margin: 5px 0;
                padding: 8px;
                border-radius: 5px;
                position: relative;
            }
            .user {
                background-color: #007bff;
                color: white;
                text-align: right;
                margin-left: 20%;
                margin-right: 5px;
            }
            .bot {
                background-color: #e9ecef;
                color: #333;
                margin-right: 20%;
                margin-left: 5px;
            }
            .timestamp {
                font-size: 0.8em;
                color: #666;
                margin-top: 5px;
            }
            .typing {
                font-style: italic;
                color: #666;
            }
            #message {
                width: 80%;
                padding: 10px;
                border: 1px solid #ccc;
                border-radius: 5px;
                font-size: 16px;
            }
            button {
                padding: 10px 20px;
                background-color: #007bff;
                color: white;
                border: none;
                border-radius: 5px;
                cursor: pointer;
                font-size: 16px;
            }
            button:hover {
                background-color: #0056b3;
            }
        </style>
    </head>
    <body>
        <h1>Constituency Issue Redressal Chatbot</h1>
        <div id="chatbox">
            <div class="message bot">Welcome! I'm here to help you report issues in your constituency. Let's get started—may I have your name, please?</div>
        </div>
        <input type="text" id="message" placeholder="Type your message here..." onkeypress="if(event.key === 'Enter') sendMessage()">
        <button onclick="sendMessage()">Send</button>

        <script>
            const chatbox = document.getElementById('chatbox');
            const messageInput = document.getElementById('message');
            let userData = {};

            function addMessage(text, sender) {
                const messageDiv = document.createElement('div');
                messageDiv.className = `message ${sender}`;
                messageDiv.textContent = text;

                const timestampDiv = document.createElement('div');
                timestampDiv.className = 'timestamp';
                const now = new Date();
                timestampDiv.textContent = now.toLocaleTimeString();
                messageDiv.appendChild(timestampDiv);

                chatbox.appendChild(messageDiv);
                chatbox.scrollTop = chatbox.scrollHeight;
            }

            function showTypingIndicator() {
                const typingDiv = document.createElement('div');
                typingDiv.className = 'message bot typing';
                typingDiv.textContent = 'Bot is typing...';
                typingDiv.id = 'typing-indicator';
                chatbox.appendChild(typingDiv);
                chatbox.scrollTop = chatbox.scrollHeight;
            }

            function removeTypingIndicator() {
                const typingDiv = document.getElementById('typing-indicator');
                if (typingDiv) typingDiv.remove();
            }

            function sendMessage() {
                const message = messageInput.value.trim();
                if (!message) return;

                addMessage(message, 'user');

                showTypingIndicator();

                fetch('http://127.0.0.1:5000/chat', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ message: message, user_data: userData })
                })
                .then(response => response.json())
                .then(data => {
                    removeTypingIndicator();
                    if (data.reply) {
                        addMessage(data.reply, 'bot');
                        userData = data.user_data || {};
                    } else if (data.error) {
                        addMessage('Error: ' + data.error, 'bot');
                    }
                })
                .catch(error => {
                    removeTypingIndicator();
                    console.error('Error:', error);
                    addMessage('Error: Could not reach the server.', 'bot');
                });

                messageInput.value = '';
            }
        </script>
    </body>
    </html>
    """)

# Dashboard Route
@app.route('/dashboard')
def dashboard():
    # Fetch data from the database
    conn = sqlite3.connect('issues.db')
    c = conn.cursor()
    c.execute("SELECT issue_category, COUNT(*) as count FROM issues GROUP BY issue_category")
    category_counts = c.fetchall()
    c.execute("SELECT * FROM issues ORDER BY timestamp DESC LIMIT 10")
    recent_issues = c.fetchall()
    conn.close()

    # Prepare data for Chart.js
    labels = [row[0] for row in category_counts]
    counts = [row[1] for row in category_counts]

    return render_template_string("""
    <!DOCTYPE html>
    <html>
    <head>
        <title>Issue Redressal Dashboard</title>
        <style>
            body {
                font-family: Arial, sans-serif;
                max-width: 1200px;
                margin: 0 auto;
                padding: 20px;
                background-color: #f4f4f4;
            }
            h1, h2 {
                text-align: center;
                color: #333;
            }
            canvas {
                max-width: 600px;
                margin: 20px auto;
            }
            table {
                width: 100%;
                border-collapse: collapse;
                margin-top: 20px;
                background-color: white;
            }
            th, td {
                padding: 10px;
                border: 1px solid #ccc;
                text-align: left;
            }
            th {
                background-color: #007bff;
                color: white;
            }
            tr:nth-child(even) {
                background-color: #f2f2f2;
            }
        </style>
        <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    </head>
    <body>
        <h1>Issue Redressal Dashboard</h1>
        <h2>Issue Categories</h2>
        <canvas id="issueChart"></canvas>
        <h2>Recent Issues</h2>
        <table>
            <tr>
                <th>ID</th>
                <th>Name</th>
                <th>Constituency</th>
                <th>Contact</th>
                <th>Issue Category</th>
                <th>Description</th>
                <th>Timestamp</th>
            </tr>
            {% for issue in recent_issues %}
            <tr>
                <td>{{ issue[0] }}</td>
                <td>{{ issue[1] }}</td>
                <td>{{ issue[2] }}</td>
                <td>{{ issue[3] }}</td>
                <td>{{ issue[4] }}</td>
                <td>{{ issue[5] }}</td>
                <td>{{ issue[6] }}</td>
            </tr>
            {% endfor %}
        </table>

        <script>
            const ctx = document.getElementById('issueChart').getContext('2d');
            new Chart(ctx, {
                type: 'bar',
                data: {
                    labels: {{ labels | tojson }},
                    datasets: [{
                        label: 'Number of Issues',
                        data: {{ counts | tojson }},
                        backgroundColor: 'rgba(0, 123, 255, 0.5)',
                        borderColor: 'rgba(0, 123, 255, 1)',
                        borderWidth: 1
                    }]
                },
                options: {
                    scales: {
                        y: {
                            beginAtZero: true,
                            title: {
                                display: true,
                                text: 'Number of Issues'
                            }
                        },
                        x: {
                            title: {
                                display: true,
                                text: 'Issue Category'
                            }
                        }
                    }
                }
            });
        </script>
    </body>
    </html>
    """, labels=labels, counts=counts, recent_issues=recent_issues)

if __name__ == "__main__":
    app.run(debug=True, use_reloader=False, port=5000)
