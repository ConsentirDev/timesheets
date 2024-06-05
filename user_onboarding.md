# User Onboarding and Usage Documentation

## Introduction

Welcome to the Python-based timesheet management application! This document will guide you through the setup, usage, and functionalities of this application. The application is built using Streamlit for the frontend and SQLite for the database. It supports two types of users: regular users and managers, each with distinct dashboard functionalities.

## Table of Contents

1. [Installation of Required Packages](#installation-of-required-packages)
2. [Database Setup](#database-setup)
3. [Running the Streamlit App](#running-the-streamlit-app)
4. [User Role Selection and Navigation](#user-role-selection-and-navigation)
5. [User Dashboard Functions](#user-dashboard-functions)
   - [Creating Timesheets](#creating-timesheets)
   - [Modifying Timesheets](#modifying-timesheets)
   - [Deleting Timesheets](#deleting-timesheets)
   - [Viewing Timesheets](#viewing-timesheets)
   - [Responding to Manager Comments](#responding-to-manager-comments)
6. [Manager Dashboard Functions](#manager-dashboard-functions)
   - [Reviewing Timesheets](#reviewing-timesheets)
   - [Managing Project Codes](#managing-project-codes)
   - [Managing Users](#managing-users)
7. [Security Considerations](#security-considerations)
8. [Conclusion](#conclusion)

## Installation of Required Packages

Install the necessary Python packages using pip. Open your terminal or command prompt and run the following command:

```sh
pip install streamlit pandas
```

Note: `sqlite3` comes pre-installed with Python, so there is no need to install it separately.

## Database Setup

The SQLite database includes tables for users, project codes, and timesheets. Please run the following script to initialize the database:

```python
import sqlite3

def init_db():
    conn = sqlite3.connect('timesheets.db')
    cursor = conn.cursor()
    
    cursor.execute('''CREATE TABLE IF NOT EXISTS users (
                        id INTEGER PRIMARY KEY,
                        username TEXT UNIQUE NOT NULL,
                        role TEXT NOT NULL CHECK(role IN ('user', 'manager'))
                      )''')
    
    cursor.execute('''CREATE TABLE IF NOT EXISTS project_codes (
                        id INTEGER PRIMARY KEY,
                        code TEXT UNIQUE NOT NULL,
                        description TEXT
                      )''')
    
    cursor.execute('''CREATE TABLE IF NOT EXISTS timesheets (
                        id INTEGER PRIMARY KEY,
                        user_id INTEGER NOT NULL,
                        project_code_id INTEGER NOT NULL,
                        date TEXT NOT NULL,
                        hours REAL NOT NULL CHECK(hours >= 0),
                        status TEXT NOT NULL CHECK(status IN ('pending', 'approved', 'rejected')),
                        comments TEXT,
                        FOREIGN KEY(user_id) REFERENCES users(id),
                        FOREIGN KEY(project_code_id) REFERENCES project_codes(id)
                      )''')

    conn.commit()
    conn.close()

init_db()
```

Run this script to set up the database before starting the application.

## Running the Streamlit App

1. Save the provided code into a Python file, e.g., `timesheet_app.py`.
2. Open a terminal or command prompt and navigate to the directory containing `timesheet_app.py`.
3. Run the Streamlit application using the following command:

```sh
streamlit run timesheet_app.py
```

The app will open in a new browser window.

## User Role Selection and Navigation

Upon launching the application, you will be presented with a navigation sidebar. Select your role (User or Manager) and enter your username. Based on your role, the appropriate dashboard will be displayed.

```python
# Welcome Screen and Navigation
st.sidebar.title("Navigation")
role = st.sidebar.selectbox("I am a...", ["User", "Manager"])
username = st.sidebar.text_input("Username")

if role == "User":
    user_id = authenticate(username, 'user')
    if user_id:
        user_dashboard(user_id)
    else:
        st.sidebar.error("Invalid user credentials")

elif role == "Manager":
    manager_id = authenticate(username, 'manager')
    if manager_id:
        manager_dashboard(manager_id)
    else:
        st.sidebar.error("Invalid manager credentials")
```

## User Dashboard Functions

### Creating Timesheets

Regular users can create new timesheets by entering the project code, date, and hours worked. The timesheet is initially marked as `pending`.

```python
def user_dashboard(user_id):
    st.title("User Dashboard")
    project_codes = [code[1] for code in get_project_codes()]

    selected_action = st.selectbox("What would you like to do?", ["Create Timesheet", "Modify Timesheet", "Delete Timesheet", "View Timesheets", "Respond to Comments"])

    if selected_action == "Create Timesheet":
        with st.form("create_timesheet", clear_on_submit=True):
            project_code = st.selectbox("Project Code", project_codes)
            dates = [st.date_input("Date", key=i) for i in range(7)]
            hours = [st.number_input(f"Hours for {dates[i].strftime('%Y-%m-%d')}", min_value=0.0, max_value=24.0) for i in range(7)]
            submitted = st.form_submit_button("Create")

            if submitted:
                project_code_id = fetch_query('SELECT id FROM project_codes WHERE code = ?', (project_code,))[0][0]
                for date, hour in zip(dates, hours):
                    run_query('INSERT INTO timesheets (user_id, project_code_id, date, hours, status) VALUES (?, ?, ?, ?, ?)',
                              (user_id, project_code_id, date.strftime('%Y-%m-%d'), hour, 'pending'))
                st.success("Timesheet created successfully!")
```

### Modifying Timesheets

Users can modify existing timesheets by entering the timesheet ID and updating the project code, date, or hours.

```python
    elif selected_action == "Modify Timesheet":
        timesheet_id = st.number_input("Enter Timesheet ID to Modify", min_value=1)
        with st.form("modify_timesheet", clear_on_submit=True):
            project_code = st.selectbox("Project Code", project_codes)
            date = st.date_input("Date")
            hours = st.number_input(f"Hours for {date.strftime('%Y-%m-%d')}", min_value=0.0, max_value=24.0)
            submitted = st.form_submit_button("Modify")

            if submitted:
                project_code_id = fetch_query('SELECT id FROM project_codes WHERE code = ?', (project_code,))[0][0]
                run_query('UPDATE timesheets SET project_code_id = ?, date = ?, hours = ? WHERE id = ? AND user_id = ?',
                          (project_code_id, date.strftime('%Y-%m-%d'), hours, timesheet_id, user_id))
                st.success(f"Timesheet {timesheet_id} modified successfully")
```

### Deleting Timesheets

Users can delete timesheets by entering the specific timesheet ID.

```python
    elif selected_action == "Delete Timesheet":
        timesheet_id = st.number_input("Enter Timesheet ID to Delete", min_value=1)
        if st.button("Delete"):
            run_query('DELETE FROM timesheets WHERE id = ? AND user_id = ?', (timesheet_id, user_id))
            st.success(f"Timesheet {timesheet_id} deleted successfully")
```

### Viewing Timesheets

Users can view all their submitted timesheets.

```python
    elif selected_action == "View Timesheets":
        timesheets = fetch_query('SELECT * FROM timesheets WHERE user_id = ?', (user_id,))
        df = pd.DataFrame(timesheets, columns=["ID", "UserID", "Project Code", "Date", "Hours", "Status", "Comments"])
        st.table(df)
```

### Responding to Manager Comments

Users can respond to manager comments on their rejected timesheets by resubmitting the timesheet with updated hours.

```python
    elif selected_action == "Respond to Comments":
        timesheets = fetch_query('SELECT id, date, status, comments FROM timesheets WHERE user_id = ? AND status = "rejected"', (user_id,))
        for timesheet in timesheets:
            st.write(f"Timesheet ID: {timesheet[0]}")
            st.write(f"Date: {timesheet[1]}")
            st.write(f"Status: {timesheet[2]}")
            st.write(f"Manager Comments: {timesheet[3]}")
            modified_hours = st.number_input(f"New Hours for {timesheet[1]}", min_value=0.0, max_value=24.0, key=f"hours_{timesheet[0]}")
            if st.button(f"Resubmit {timesheet[0]}", key=f"resubmit_{timesheet[0]}"):
                run_query('UPDATE timesheets SET hours = ?, status = "pending" WHERE id = ?', (modified_hours, timesheet[0]))
                st.success(f"Timesheet {timesheet[0]} resubmitted")
```

## Manager Dashboard Functions

### Reviewing Timesheets

Managers can review pending timesheets, add comments, and change their status to `approved` or `rejected`.

```python
def manager_dashboard(manager_id):
    st.title("Manager Dashboard")
    selected_action = st.selectbox("What would you like to do?", ["Review Timesheets", "Manage Project Codes", "Manage Users"])

    if selected_action == "Review Timesheets":
        timesheets = fetch_query('SELECT * FROM timesheets WHERE status = "pending"')
        for timesheet in timesheets:
            st.write(f"Timesheet ID: {timesheet[0]}")
            st.write(f"User ID: {timesheet[1]}")
            st.write(f"Project Code ID: {timesheet[2]}")
            st.write(f"Date: {timesheet[3]}")
            st.write(f"Hours: {timesheet[4]}")
            comments = st.text_area(f"Comments for Timesheet {timesheet[0]}")
            if st.button(f"Approve {timesheet[0]}", key=f"approve_{timesheet[0]}"):
                run_query('UPDATE timesheets SET status = "approved", comments = ? WHERE id = ?', (comments, timesheet[0]))
                st.success(f"Timesheet {timesheet[0]} approved")
            if st.button(f"Reject {timesheet[0]}", key=f"reject_{timesheet[0]}"):
                run_query('UPDATE timesheets SET status = "rejected", comments = ? WHERE id = ?', (comments, timesheet[0]))
                st.success(f"Timesheet {timesheet[0]} rejected")
```

### Managing Project Codes

Managers can create, modify, or delete project codes within the application.

```python
    elif selected_action == "Manage Project Codes":
        project_codes = fetch_query('SELECT * FROM project_codes')
        st.table(pd.DataFrame(project_codes, columns=["ID", "Code", "Description"]))

        with st.form("manage_project_codes", clear_on_submit=True):
            action = st.selectbox("Action", ["Create", "Modify", "Delete"])
            code = st.text_input("Project Code")
            description = st.text_input("Description")
            
            project_code_id = None
            if action in ["Modify", "Delete"]:
                project_code_id = st.number_input("Enter Project Code ID to Modify/Delete", min_value=1)
            
            submitted = st.form_submit_button(f"{action} Project Code")
            if submitted:
                if action == "Create":
                    run_query('INSERT INTO project_codes (code, description) VALUES (?, ?)', (code, description))
                    st.success("Project Code created successfully!")
                elif action == "Modify" and project_code_id:
                    run_query('UPDATE project_codes SET code = ?, description = ? WHERE id = ?', (code, description, project_code_id))
                    st.success(f"Project Code {project_code_id} modified successfully")
                elif action == "Delete" and project_code_id:
                    run_query('DELETE FROM project_codes WHERE id = ?', (project_code_id,))
                    st.success(f"Project Code {project_code_id} deleted successfully")
```

### Managing Users

Managers can create, modify, or delete users within the application.

```python
    elif selected_action == "Manage Users":
        users = fetch_query('SELECT * FROM users')
        st.table(pd.DataFrame(users, columns=["ID", "Username", "Role"]))

        with st.form("manage_users", clear_on_submit=True):
            action = st.selectbox("Action", ["Create", "Modify", "Delete"])
            username = st.text_input("Username")
            role = st.selectbox("Role", ["user", "manager"])
            
            user_id = None
            if action in ["Modify", "Delete"]:
                user_id = st.number_input("Enter User ID to Modify/Delete", min_value=1)
            
            submitted = st.form_submit_button(f"{action} User")
            if submitted:
                if action == "Create":
                    run_query('INSERT INTO users (username, role) VALUES (?, ?)', (username, role))
                    st.success("User created successfully!")
                elif action == "Modify" and user_id:
                    run_query('UPDATE users SET username = ?, role = ? WHERE id = ?', (username, role, user_id))
                    st.success(f"User {username} modified successfully")
                elif action == "Delete" and user_id:
                    run_query('DELETE FROM users WHERE id = ?', (user_id,))
                    st.success(f"User {username} deleted successfully")
```

## Security Considerations

- **Authentication:** Basic authentication is implemented. Enhance security by incorporating proper login mechanisms and password hashing (e.g., using `bcrypt` or `Argon2`).
- **Input Validation:** Ensure input sanitization to prevent SQL Injection. Use parameterized queries.
- **Error Handling:** Improve error handling for database interactions to enhance robustness.

## Conclusion

This application is designed to be a user-friendly timesheet management system leveraging Streamlit for the frontend and SQLite for the backend. Further refinements in user experience, database design, and security mechanisms can be made as per specific requirements.

Thank you for using the timesheet management application! If you have any questions or need further assistance, please refer to the code documentation provided or contact your system administrator. Happy time tracking!