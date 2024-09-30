# Updated Pilot Implementation with PostgreSQL Backend and OpenWeb UI

In this updated version, we enhance the previous pilot implementation by integrating a PostgreSQL backend for data storage and an OpenWeb UI for user interaction. This setup allows for persistent data management and provides a user-friendly interface to interact with the agents, monitor progress, and manage projects.

---

## **Overview**

- **Backend**: PostgreSQL database to store agent outputs, user stories, implementations, feedback, and project states.
- **Frontend**: OpenWeb UI built using Flask (a Python web framework) for initiating projects, displaying agent interactions, and controlling the workflow.
- **Agents**: Utilize the OpenRouter API for LLM interactions, with agents belonging to different LLMs, including assistant and verification agents.
- **Process Flow**:
  1. User starts a new project via the OpenWeb UI.
  2. PM Agent creates user stories and saves them to the database.
  3. Assistant PM Agent reviews and refines user stories.
  4. Dev Agent implements features based on refined user stories.
  5. Assistant Dev Agent reviews the implementation.
  6. Verification Agents independently verify steps and provide feedback.
  7. User can view all outputs and initiate deployment via the OpenWeb UI.
  8. Deployment is simulated, and results are displayed.

---

## **Implementation Steps**

1. **Set Up the Environment**
2. **Configure PostgreSQL Backend**
3. **Develop the OpenWeb UI**
4. **Update Agent Functions for Database Integration**
5. **Implement Agent Interactions and Workflow**
6. **Run and Test the Application**

---

## **1. Set Up the Environment**

### **Prerequisites**

- **Python 3.7+**
- **OpenRouter API Key**: Sign up at [OpenRouter](https://openrouter.ai/) to obtain an API key.
- **PostgreSQL Database**: Install and set up PostgreSQL.
- **Required Python Libraries**:
  - `requests` for API calls
  - `psycopg2-binary` for PostgreSQL interaction
  - `SQLAlchemy` for ORM
  - `Flask` for the web framework
  - `Jinja2` for templating (included with Flask)
  - `python-dotenv` for environment variables

### **Install Required Libraries**

```bash
pip install requests psycopg2-binary SQLAlchemy Flask python-dotenv
```

---

## **2. Configure PostgreSQL Backend**

### **Install PostgreSQL**

Download and install PostgreSQL from the [official website](https://www.postgresql.org/download/) or use your package manager.

### **Set Up the Database**

1. **Create a Database User and Database**

```bash
# Access PostgreSQL shell
sudo -u postgres psql

# Create a user
CREATE USER myuser WITH PASSWORD 'mypassword';

# Create a database
CREATE DATABASE mydatabase;

# Grant privileges
GRANT ALL PRIVILEGES ON DATABASE mydatabase TO myuser;

# Exit
\q
```

2. **Define Database Models**

Create a `models.py` file to define your database schema using SQLAlchemy ORM.

```python
# models.py
from sqlalchemy import Column, Integer, String, Text, DateTime, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
import datetime

Base = declarative_base()

class Project(Base):
    __tablename__ = 'projects'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)

class UserStory(Base):
    __tablename__ = 'user_stories'
    id = Column(Integer, primary_key=True)
    project_id = Column(Integer, ForeignKey('projects.id'))
    content = Column(Text)
    improved_content = Column(Text)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)

class Implementation(Base):
    __tablename__ = 'implementations'
    id = Column(Integer, primary_key=True)
    project_id = Column(Integer, ForeignKey('projects.id'))
    content = Column(Text)
    improved_content = Column(Text)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)

class AgentFeedback(Base):
    __tablename__ = 'agent_feedback'
    id = Column(Integer, primary_key=True)
    project_id = Column(Integer, ForeignKey('projects.id'))
    step = Column(String)
    content = Column(Text)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)

class DeploymentStatus(Base):
    __tablename__ = 'deployment_status'
    id = Column(Integer, primary_key=True)
    project_id = Column(Integer, ForeignKey('projects.id'))
    status = Column(String)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)
```

3. **Create Tables**

Create a `database.py` file to set up the engine and session.

```python
# database.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from models import Base
import os

DATABASE_URL = os.getenv('DATABASE_URL', 'postgresql://myuser:mypassword@localhost/mydatabase')

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)

def init_db():
    Base.metadata.create_all(bind=engine)
```

Run the following script to initialize the database:

```python
# initialize_db.py
from database import init_db

if __name__ == '__main__':
    init_db()
```

---

## **3. Develop the OpenWeb UI**

### **Set Up Flask Application**

Create an `app.py` file.

```python
# app.py
from flask import Flask, render_template, request, redirect, url_for
from database import SessionLocal
from models import Project, UserStory, Implementation, AgentFeedback, DeploymentStatus

app = Flask(__name__)
```

### **Templates**

Create a `templates` directory with the following HTML files:

#### **`index.html`**

```html
<!doctype html>
<html>
<head>
    <title>Two-Agent LLM SDLC Pilot</title>
</head>
<body>
    <h1>Two-Agent LLM SDLC Pilot</h1>
    <form action="/start_project" method="post">
        <label for="project_name">Project Name:</label>
        <input type="text" name="project_name" required>
        <button type="submit">Start Project</button>
    </form>
</body>
</html>
```

#### **`project.html`**

```html
<!doctype html>
<html>
<head>
    <title>Project {{ project.name }}</title>
</head>
<body>
    <h1>Project: {{ project.name }}</h1>

    <h2>User Stories</h2>
    <pre>{{ user_story.content }}</pre>

    <h2>Improved User Stories</h2>
    <pre>{{ user_story.improved_content }}</pre>

    <h2>Implementation</h2>
    <pre>{{ implementation.content }}</pre>

    <h2>Improved Implementation</h2>
    <pre>{{ implementation.improved_content }}</pre>

    <h2>User Stories Feedback</h2>
    <pre>{{ user_stories_feedback }}</pre>

    <h2>Implementation Feedback</h2>
    <pre>{{ implementation_feedback }}</pre>

    {% if deployment_status is none %}
    <form action="/deploy/{{ project.id }}" method="post">
        <button type="submit">Deploy Implementation</button>
    </form>
    {% else %}
    <h2>Deployment Status</h2>
    <p>{{ deployment_status.status }}</p>
    {% endif %}
</body>
</html>
```

---

## **4. Update Agent Functions for Database Integration**

Update agent functions to interact with the database.

```python
import os
import random
import datetime
import requests
from sqlalchemy.orm import scoped_session
from flask import g

# Load API key from environment variable
OPENROUTER_API_KEY = os.getenv('OPENROUTER_API_KEY')
OPENROUTER_API_URL = 'https://openrouter.ai/api/v1/chat/completions'

def call_openrouter_api(messages, model):
    headers = {
        'Authorization': f'Bearer {OPENROUTER_API_KEY}',
        'Content-Type': 'application/json',
        'X-Title': 'Two-Agent LLM SDLC Pilot'
    }
    data = {
        'model': model,
        'messages': messages
    }
    response = requests.post(OPENROUTER_API_URL, headers=headers, json=data)
    response.raise_for_status()
    return response.json()['choices'][0]['message']['content']
```

### **Database Session Management**

```python
from flask import g

def get_db():
    if 'db' not in g:
        g.db = scoped_session(SessionLocal)
    return g.db

@app.teardown_appcontext
def teardown_db(exception):
    db = g.pop('db', None)
    if db is not None:
        db.remove()
```

### **Agent Functions**

#### **PM Agent Creates User Stories**

```python
def pm_agent_create_user_stories(db, project_id):
    pm_agent_prompt = """
    You are the PM Agent responsible for creating and prioritizing user stories, defining acceptance criteria, and reviewing implemented features.
    """
    messages = [
        {'role': 'system', 'content': pm_agent_prompt},
        {'role': 'user', 'content': 'Create user stories for a to-do list application.'}
    ]
    model = 'openai/gpt-4'
    user_stories = call_openrouter_api(messages, model)

    # Save to database
    user_story = UserStory(project_id=project_id, content=user_stories, created_at=datetime.datetime.utcnow())
    db.add(user_story)
    db.commit()
    return user_story
```

#### **Assistant PM Agent Reviews User Stories**

```python
def assistant_pm_agent_review(db, user_story):
    assistant_pm_agent_prompt = """
    You are the Assistant PM Agent. Review the user stories created by the PM Agent, suggest improvements, and ensure clarity and completeness.
    """
    assistant_models = [
        'openai/gpt-3.5-turbo',
        'anthropic/claude-instant-v1',
        'togethercomputer/llama-2-7b-chat',
        'cohere/command-medium-nightly'
    ]
    model = random.choice(assistant_models)
    messages = [
        {'role': 'system', 'content': assistant_pm_agent_prompt},
        {'role': 'user', 'content': f'Review the following user stories and suggest improvements:\n\n{user_story.content}'}
    ]
    improved_user_stories = call_openrouter_api(messages, model)

    # Update in database
    user_story.improved_content = improved_user_stories
    user_story.updated_at = datetime.datetime.utcnow()
    db.commit()
    return improved_user_stories
```

Similarly, implement the `dev_agent_implement_features`, `assistant_dev_agent_review`, and `verification_agent_verify` functions, updating the database accordingly.

---

## **5. Implement Agent Interactions and Workflow**

### **Route Handlers in Flask**

#### **Index Route**

```python
@app.route('/')
def index():
    return render_template('index.html')
```

#### **Start Project Route**

```python
@app.route('/start_project', methods=['POST'])
def start_project():
    project_name = request.form['project_name']
    db = get_db()

    # Create new project
    project = Project(name=project_name)
    db.add(project)
    db.commit()

    # Start the workflow
    user_story = pm_agent_create_user_stories(db, project.id)
    assistant_pm_agent_review(db, user_story)
    implementation = dev_agent_implement_features(db, project.id, user_story)
    assistant_dev_agent_review(db, implementation)
    user_stories_feedback, implementation_feedback = run_verification_steps(db, project.id, user_story, implementation)

    return redirect(url_for('view_project', project_id=project.id))
```

#### **View Project Route**

```python
@app.route('/project/<int:project_id>')
def view_project(project_id):
    db = get_db()
    project = db.query(Project).get(project_id)
    user_story = db.query(UserStory).filter_by(project_id=project_id).first()
    implementation = db.query(Implementation).filter_by(project_id=project_id).first()
    deployment_status = db.query(DeploymentStatus).filter_by(project_id=project_id).first()

    # Retrieve feedback
    user_stories_feedback = db.query(AgentFeedback).filter_by(project_id=project_id, step='user_stories').first()
    implementation_feedback = db.query(AgentFeedback).filter_by(project_id=project_id, step='implementation').first()

    return render_template('project.html', project=project, user_story=user_story,
                           implementation=implementation,
                           user_stories_feedback=user_stories_feedback.content if user_stories_feedback else '',
                           implementation_feedback=implementation_feedback.content if implementation_feedback else '',
                           deployment_status=deployment_status)
```

#### **Deploy Project Route**

```python
@app.route('/deploy/<int:project_id>', methods=['POST'])
def deploy_project(project_id):
    db = get_db()
    deployment_status = DeploymentStatus(project_id=project_id, status='Deployment Successful', created_at=datetime.datetime.utcnow())
    db.add(deployment_status)
    db.commit()
    return redirect(url_for('view_project', project_id=project_id))
```

---

## **6. Run and Test the Application**

### **Run the Flask Application**

```python
if __name__ == '__main__':
    app.run(debug=True)
```

**Note**: In a production environment, disable debug mode and use a production-ready server like Gunicorn or uWSGI.

### **Test the Application**

- Start the Flask application:

  ```bash
  python app.py
  ```

- Open your browser and navigate to `http://localhost:5000/`.

- Start a new project by entering a project name.

- The application will run through the workflow, and you can view the results on the project page.

---

## **Security Considerations**

- **API Keys**: Store your OpenRouter API key and database credentials securely using environment variables or a `.env` file (remember to add `.env` to `.gitignore`).

  ```bash
  # .env file
  OPENROUTER_API_KEY=your_api_key
  DATABASE_URL=postgresql://myuser:mypassword@localhost/mydatabase
  ```

  Load environment variables in your application:

  ```python
  from dotenv import load_dotenv
  load_dotenv()
  ```

- **Input Validation**: Ensure all user inputs are properly validated and sanitized to prevent injection attacks.

- **Error Handling**: Implement try-except blocks around API calls and database operations to handle exceptions gracefully.

- **HTTPS**: Use HTTPS in production to encrypt data in transit.

- **Authentication**: Implement user authentication and authorization if the application is exposed to multiple users.

---

## **Additional Enhancements**

- **Asynchronous Operations**: Use `asyncio` and `aiohttp` to make asynchronous API calls to improve performance.

- **Logging**: Implement logging using Python's `logging` module to track application behavior.

- **Frontend Improvements**: Use a frontend framework like React or Vue.js for a more dynamic user interface.

- **Containerization**: Dockerize the application for easier deployment and scalability.

- **Continuous Integration/Continuous Deployment (CI/CD)**: Set up a CI/CD pipeline using tools like Jenkins or GitHub Actions.

---

## **Conclusion**

This updated implementation integrates a PostgreSQL backend and an OpenWeb UI, enhancing the functionality and usability of the Two-Agent LLM SDLC Method. By storing data persistently, we enable tracking, auditing, and collaboration across the software development lifecycle. The OpenWeb UI provides an accessible interface for users to initiate projects, monitor progress, and interact with agent outputs.

---

## **Next Steps**

- **Expand Agent Functionality**: Add more detailed prompts and handling for complex tasks.

- **Implement Human Oversight**: Introduce features that allow human supervisors to review and intervene at various stages.

- **Scalability**: Optimize the application for handling multiple concurrent projects and users.

- **Monitoring and Analytics**: Implement monitoring tools to track performance metrics and agent effectiveness.

---

## **Disclaimer**

Ensure compliance with the terms of service of OpenRouter and the respective LLM providers when using their APIs. Be cautious about data privacy, security, and usage policies.

---

## **References**

- [Flask Documentation](https://flask.palletsprojects.com/)
- [SQLAlchemy Documentation](https://docs.sqlalchemy.org/)
- [OpenRouter API Documentation](https://openrouter.ai/docs)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

---

**Note**: Replace placeholder values like `'your_api_key'`, `'myuser'`, `'mypassword'`, and `'mydatabase'` with your actual credentials. Always keep sensitive information secure and avoid hardcoding credentials in your codebase.
