# Smart-attendance-system-
import sqlite3
from datetime import datetime

# -------------------------- DATABASE SETUP --------------------------
def init_database():
    """Create all necessary tables as per Chapter 4: System Design"""
    conn = sqlite3.connect('attendance_system.db')
    cursor = conn.cursor()

    # 1. Users Table (Admin/Staff/Lecturer)
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS Users (
        user_id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE NOT NULL,
        password TEXT NOT NULL,
        role TEXT NOT NULL CHECK(role IN ('admin', 'staff', 'lecturer'))
    )
    ''')

    # 2. Students Table
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS Students (
        student_id TEXT PRIMARY KEY,
        name TEXT NOT NULL,
        course TEXT NOT NULL,
        class_name TEXT NOT NULL
    )
    ''')

    # 3. Sessions/Classes Table
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS Sessions (
        session_id INTEGER PRIMARY KEY AUTOINCREMENT,
        session_name TEXT NOT NULL,
        date TEXT NOT NULL
    )
    ''')

    # 4. Attendance Table
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS Attendance (
        attendance_id INTEGER PRIMARY KEY AUTOINCREMENT,
        student_id TEXT NOT NULL,
        session_id INTEGER NOT NULL,
        status TEXT NOT NULL CHECK(status IN ('Present', 'Absent')),
        time_marked TEXT NOT NULL,
        FOREIGN KEY (student_id) REFERENCES Students(student_id),
        FOREIGN KEY (session_id) REFERENCES Sessions(session_id)
    )
    ''')

    # Insert default admin account
    try:
        cursor.execute("INSERT INTO Users (username, password, role) VALUES (?, ?, ?)",
                       ("admin", "admin123", "admin"))
    except sqlite3.IntegrityError:
        pass  # Already exists

    conn.commit()
    conn.close()
    print("✅ Database initialized successfully.")

# -------------------------- CORE SYSTEM CLASS --------------------------
class SmartAttendanceSystem:
    def __init__(self):
        self.logged_in_user = None
        self.init_db = init_database()

    # --- USER MODULE ---
    def login(self, username, password):
        """Authenticate user (Login Module)"""
        conn = sqlite3.connect('attendance_system.db')
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM Users WHERE username = ? AND password = ?", (username, password))
        user = cursor.fetchone()
        conn.close()
        if user:
            self.logged_in_user = {"id": user[0], "username": user[1], "role": user[3]}
            return True
        return False

    # --- STUDENT MANAGEMENT MODULE ---
    def add_student(self, student_id, name, course, class_name):
        """Add new student record"""
        try:
            conn = sqlite3.connect('attendance_system.db')
            cursor = conn.cursor()
            cursor.execute("INSERT INTO Students VALUES (?, ?, ?, ?)",
                          (student_id, name, course, class_name))
            conn.commit()
            conn.close()
            return True
        except sqlite3.IntegrityError:
            return False  # Duplicate ID

    def view_all_students(self):
        """Display all registered students"""
        conn = sqlite3.connect('attendance_system.db')
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM Students")
        students = cursor.fetchall()
        conn.close()
        return students

    # --- SESSION MODULE ---
    def create_session(self, session_name, date=None):
        """Create new class/session"""
        if not date:
            date = datetime.now().strftime("%Y-%m-%d")
        conn = sqlite3.connect('attendance_system.db')
        cursor = conn.cursor()
        cursor.execute("INSERT INTO Sessions (session_name, date) VALUES (?, ?)",
                      (session_name, date))
        conn.commit()
        session_id = cursor.lastrowid
        conn.close()
        return session_id

    def view_sessions(self):
        """View all sessions"""
        conn = sqlite3.connect('attendance_system.db')
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM Sessions ORDER BY date DESC")
        sessions = cursor.fetchall()
        conn.close()
        return sessions

    # --- ATTENDANCE MODULE ---
    def mark_attendance(self, session_id, student_id, status):
        """Mark student present or absent"""
        time_now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        conn = sqlite3.connect('attendance_system.db')
        cursor = conn.cursor()
        # Check if already marked
        cursor.execute("SELECT * FROM Attendance WHERE session_id = ? AND student_id = ?",
                      (session_id, student_id))
        if cursor.fetchone():
            conn.close()
            return False, "⚠️ Attendance already marked for this student."

        cursor.execute("INSERT INTO Attendance VALUES (NULL, ?, ?, ?, ?)",
                      (student_id, session_id, status, time_now))
        conn.commit()
        conn.close()
        return True, "✅ Attendance marked successfully."

    # --- REPORT MODULE ---
    def generate_report(self, session_id=None):
        """Generate attendance reports (Chapter 4 & 6)"""
        conn = sqlite3.connect('attendance_system.db')
        cursor = conn.cursor()

        if session_id:
            # Report for specific session
            cursor.execute('''
            SELECT s.student_id, s.name, s.class_name, a.status, a.time_marked
            FROM Attendance a
            JOIN Students s ON a.student_id = s.student_id
            WHERE a.session_id = ?
            ''', (session_id,))
        else:
            # All records
            cursor.execute('''
            SELECT s.student_id, s.name, se.session_name, se.date, a.status
            FROM Attendance a
            JOIN Students s ON a.student_id = s.student_id
            JOIN Sessions se ON a.session_id = se.session_id
            ORDER BY se.date DESC
            ''')

        report_data = cursor.fetchall()
        conn.close()
        return report_data

# -------------------------- MENU INTERFACE --------------------------
def main_menu():
    system = SmartAttendanceSystem()
    init_database()

    print("="*60)
    print("📚 SMART ATTENDANCE MANAGEMENT SYSTEM - KIBOI NATIONAL POLYTECHNIC")
    print("👨‍💻 Developed by: Brian Kimutai Rono | Diploma in ICT")
    print("="*60)

    # Login Screen
    while True:
        print("\n🔐 LOGIN")
        user = input("Username: ")
        pwd = input("Password: ")
        if system.login(user, pwd):
            print(f"✅ Welcome! Logged in as {system.logged_in_user['role'].upper()}")
            break
        else:
            print("❌ Invalid credentials. Try again.")

    # Main Application Loop
    while True:
        print("\n" + "-"*50)
        print("🏠 MAIN MENU")
        print("1. 📝 Add New Student")
        print("2. 📋 View All Students")
        print("3. 📅 Create New Session/Class")
        print("4. ✅ Mark Attendance")
        print("5. 📊 Generate Attendance Report")
        print("6. 🚪 Exit")
        print("-"*50)

        choice = input("Enter your choice (1-6): ")

        if choice == '1':
            print("\n--- Add New Student ---")
            sid = input("Student ID: ")
            name = input("Full Name: ")
            course = input("Course: ")
            cls = input("Class/Stream: ")
            if system.add_student(sid, name, course, cls):
                print("✅ Student added successfully!")
            else:
                print("❌ Error: Student ID already exists.")

        elif choice == '2':
            print("\n--- Student List ---")
            students = system.view_all_students()
            if students:
                for s in students:
                    print(f"ID: {s[0]:<10} | Name: {s[1]:<20} | Course: {s[2]:<15} | Class: {s[3]}")
            else:
                print("⚠️ No students registered.")

        elif choice == '3':
            print("\n--- Create Session ---")
            name = input("Session Name (e.g., ICT Year 1): ")
            session_id = system.create_session(name)
            print(f"✅ Session created with ID: {session_id}")

        elif choice == '4':
            print("\n--- Mark Attendance ---")
            sessions = system.view_sessions()
            if not sessions:
                print("⚠️ Create a session first.")
                continue
            print("Available Sessions:")
            for se in sessions:
                print(f"ID: {se[0]} | {se[1]} | Date: {se[2]}")
            sid = int(input("Enter Session ID: "))

            students = system.view_all_students()
            if not students:
                print("⚠️ Add students first.")
                continue

            for stu in students:
                status = input(f"Mark {stu[1]} (ID: {stu[0]}) as [P]resent / [A]bsent: ").upper()
                stat = "Present" if status == 'P' else "Absent"
                msg = system.mark_attendance(sid, stu[0], stat)
                print(msg[1])

        elif choice == '5':
            print("\n--- Attendance Report ---")
            reports = system.generate_report()
            if reports:
                print(f"{'ID':<10} | {'Name':<20} | {'Session':<15} | {'Date':<10} | {'Status'}")
                print("-"*70)
                for r in reports:
                    print(f"{r[0]:<10} | {r[1]:<20} | {r[2]:<15} | {r[3]:<10} | {r[4]}")
            else:
                print("⚠️ No attendance records found.")

        elif choice == '6':
            print("👋 Exiting system. Goodbye!")
            break

        else:
            print("❌ Invalid choice. Please try again.")

# -------------------------- RUN PROGRAM --------------------------
if __name__ == "__main__":
    main_menu()
    
