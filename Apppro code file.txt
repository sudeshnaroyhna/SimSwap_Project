"""
==============================================================================
SIM SWAP FRAUD DETECTION SYSTEM
DBMS Mini-Project Application (Tkinter + SQLite3)
==============================================================================
Description:
    A complete desktop application designed for telecom fraud monitoring.
    Analyzes SIM replacement requests, calculates risk scores using custom
    rule-based logic, generates fraud alerts, and stores records in SQLite.
    Includes advanced theme customization (12 Default Themes + Custom Theme Creator)
    and pre-loaded demo data.

Execution:
    python main.py
==============================================================================
"""

import sqlite3
import csv
import os
from datetime import datetime
import tkinter as tk
from tkinter import ttk, messagebox, filedialog, colorchooser

# ============================================================================
# DATABASE MANAGER CLASS
# ============================================================================
class DatabaseManager:
    """Handles all SQLite database initialization, queries, and record storage."""
    
    def __init__(self, db_name="sim_swap.db"):
        self.db_name = db_name
        self.init_db()

    def get_connection(self):
        """Returns a new database connection object."""
        return sqlite3.connect(self.db_name)

    def init_db(self):
        """Creates tables and populates 5 sample demo records if newly created."""
        with self.get_connection() as conn:
            cursor = conn.cursor()
            
            # 1. Customers Table
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS customers (
                    customer_id INTEGER PRIMARY KEY AUTOINCREMENT,
                    name TEXT NOT NULL,
                    mobile_number TEXT NOT NULL,
                    aadhaar_number TEXT NOT NULL
                )
            ''')

            # 2. SIM Requests Table
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS sim_requests (
                    request_id INTEGER PRIMARY KEY AUTOINCREMENT,
                    customer_id INTEGER,
                    request_date TEXT NOT NULL,
                    location TEXT NOT NULL,
                    reason TEXT NOT NULL,
                    risk_score INTEGER NOT NULL,
                    risk_level TEXT NOT NULL,
                    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
                )
            ''')

            # 3. Alerts Table
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS alerts (
                    alert_id INTEGER PRIMARY KEY AUTOINCREMENT,
                    request_id INTEGER,
                    message TEXT NOT NULL,
                    risk_score INTEGER NOT NULL,
                    risk_level TEXT NOT NULL,
                    timestamp TEXT NOT NULL,
                    FOREIGN KEY (request_id) REFERENCES sim_requests(request_id)
                )
            ''')

            # 4. Fraud Incidents Table
            cursor.execute('''
                CREATE TABLE IF NOT EXISTS fraud_incidents (
                    incident_id INTEGER PRIMARY KEY AUTOINCREMENT,
                    alert_id INTEGER,
                    status TEXT NOT NULL,
                    created_at TEXT NOT NULL,
                    FOREIGN KEY (alert_id) REFERENCES alerts(alert_id)
                )
            ''')
            conn.commit()

            # Seed Demo Data if Database is empty
            cursor.execute("SELECT COUNT(*) FROM sim_requests")
            if cursor.fetchone()[0] == 0:
                self.seed_demo_data()

    def seed_demo_data(self):
        """Populates 5 sample demo records representing different risk profiles."""
        demo_records = [
            ("Rajesh Kumar", "9876543210", "[Aadhaar Omitted]", "2021-05-10", "2026-09-15", "Mumbai", "Damaged SIM", 0, "Low Risk"),
            ("Anita Sharma", "9123456789", "[Aadhaar Omitted]", "2022-11-20", "2026-09-18", "Delhi", "Lost Phone", 20, "Low Risk"),
            ("Vikram Singh", "9988776655", "[Aadhaar Omitted]", "2023-01-15", "2026-09-20", "Bangalore", "4G to 5G Upgrade", 45, "Medium Risk"),
            ("Suresh Patel", "9811223344", "[Aadhaar Omitted]", "2024-03-01", "2026-09-22", "Kolkata", "SIM Card Corrupted", 75, "High Risk"),
            ("Priya Nair", "9744556677", "[Aadhaar Omitted]", "2025-08-12", "2026-09-25", "Chennai", "Lost Phone", 100, "Critical")
        ]

        for name, mobile, aadhaar, prev_d, curr_d, loc, reason, score, level in demo_records:
            self.insert_sim_swap_transaction(name, mobile, aadhaar, prev_d, curr_d, loc, reason, score, level)

    def insert_sim_swap_transaction(self, name, mobile, aadhaar, prev_date, curr_date, location, reason, risk_score, risk_level):
        """Processes and stores a SIM Swap request and auto-generates alerts if score > 50."""
        with self.get_connection() as conn:
            cursor = conn.cursor()

            # Insert Customer Record
            cursor.execute('''
                INSERT INTO customers (name, mobile_number, aadhaar_number)
                VALUES (?, ?, ?)
            ''', (name, mobile, aadhaar))
            customer_id = cursor.lastrowid

            # Insert Request Record
            cursor.execute('''
                INSERT INTO sim_requests (customer_id, request_date, location, reason, risk_score, risk_level)
                VALUES (?, ?, ?, ?, ?, ?)
            ''', (customer_id, curr_date, location, reason, risk_score, risk_level))
            request_id = cursor.lastrowid

            # Rule: Generate Alert if Risk Score > 50
            if risk_score > 50:
                timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
                message = f"High Risk SIM Swap detected for {name} ({mobile}). Score: {risk_score}"

                cursor.execute('''
                    INSERT INTO alerts (request_id, message, risk_score, risk_level, timestamp)
                    VALUES (?, ?, ?, ?, ?)
                ''', (request_id, message, risk_score, risk_level, timestamp))
                alert_id = cursor.lastrowid

                # Store as Fraud Incident
                cursor.execute('''
                    INSERT INTO fraud_incidents (alert_id, status, created_at)
                    VALUES (?, ?, ?)
                ''', (alert_id, "New", timestamp))

            conn.commit()

    def get_dashboard_stats(self):
        """Fetches aggregate numbers for dashboard metrics."""
        with self.get_connection() as conn:
            cursor = conn.cursor()
            
            cursor.execute("SELECT COUNT(*) FROM sim_requests")
            total = cursor.fetchone()[0]

            cursor.execute("SELECT COUNT(*) FROM sim_requests WHERE risk_level = 'Low Risk'")
            safe = cursor.fetchone()[0]

            cursor.execute("SELECT COUNT(*) FROM sim_requests WHERE risk_level = 'Medium Risk'")
            medium = cursor.fetchone()[0]

            cursor.execute("SELECT COUNT(*) FROM sim_requests WHERE risk_level = 'High Risk'")
            high = cursor.fetchone()[0]

            cursor.execute("SELECT COUNT(*) FROM sim_requests WHERE risk_level = 'Critical'")
            critical = cursor.fetchone()[0]

            cursor.execute("SELECT COUNT(*) FROM fraud_incidents")
            incidents = cursor.fetchone()[0]

            return {
                "total": total,
                "safe": safe,
                "medium": medium,
                "high": high + critical,
                "incidents": incidents
            }

    def get_recent_alerts(self, limit=10):
        """Retrieves recent alerts with associated customer details."""
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                SELECT a.alert_id, c.name, c.mobile_number, a.risk_score, a.risk_level, fi.status, a.timestamp
                FROM alerts a
                JOIN sim_requests sr ON a.request_id = sr.request_id
                JOIN customers c ON sr.customer_id = c.customer_id
                LEFT JOIN fraud_incidents fi ON a.alert_id = fi.alert_id
                ORDER BY a.alert_id DESC
                LIMIT ?
            ''', (limit,))
            return cursor.fetchall()

    def get_all_incidents(self):
        """Retrieves all logged fraud incidents."""
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute('''
                SELECT fi.incident_id, c.name, a.risk_score, a.risk_level, fi.status, fi.created_at
                FROM fraud_incidents fi
                JOIN alerts a ON fi.alert_id = a.alert_id
                JOIN sim_requests sr ON a.request_id = sr.request_id
                JOIN customers c ON sr.customer_id = c.customer_id
                ORDER BY fi.incident_id DESC
            ''')
            return cursor.fetchall()

    def update_incident_status(self, incident_id, new_status):
        """Updates the status of a fraud incident."""
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute("UPDATE fraud_incidents SET status = ? WHERE incident_id = ?", (new_status, incident_id))
            conn.commit()

    def delete_incident(self, incident_id):
        """Deletes an incident record."""
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute("DELETE FROM fraud_incidents WHERE incident_id = ?", (incident_id,))
            conn.commit()

    def get_database_records(self, search_term="", risk_filter="All"):
        """Fetches joined table records with optional search and filtering."""
        with self.get_connection() as conn:
            cursor = conn.cursor()
            query = '''
                SELECT sr.request_id, c.name, c.mobile_number, c.aadhaar_number, 
                       sr.location, sr.reason, sr.risk_score, sr.risk_level, sr.request_date
                FROM sim_requests sr
                JOIN customers c ON sr.customer_id = c.customer_id
                WHERE (c.name LIKE ? OR c.mobile_number LIKE ?)
            '''
            params = [f"%{search_term}%", f"%{search_term}%"]

            if risk_filter != "All":
                query += " AND sr.risk_level = ?"
                params.append(risk_filter)

            query += " ORDER BY sr.request_id DESC"
            cursor.execute(query, params)
            return cursor.fetchall()

    def get_report_data(self, report_type):
        """Generates datasets for reporting views and CSV exportation."""
        with self.get_connection() as conn:
            cursor = conn.cursor()
            if report_type == "Total Requests":
                cursor.execute('''
                    SELECT sr.request_id, c.name, c.mobile_number, sr.location, sr.risk_score, sr.risk_level, sr.request_date
                    FROM sim_requests sr JOIN customers c ON sr.customer_id = c.customer_id
                ''')
                headers = ["Request ID", "Name", "Mobile Number", "Location", "Risk Score", "Risk Level", "Date"]
            elif report_type == "Fraud Incidents":
                cursor.execute('''
                    SELECT fi.incident_id, c.name, c.mobile_number, a.risk_score, a.risk_level, fi.status, fi.created_at
                    FROM fraud_incidents fi
                    JOIN alerts a ON fi.alert_id = a.alert_id
                    JOIN sim_requests sr ON a.request_id = sr.request_id
                    JOIN customers c ON sr.customer_id = c.customer_id
                ''')
                headers = ["Incident ID", "Name", "Mobile Number", "Risk Score", "Risk Level", "Status", "Created At"]
            elif report_type == "Risk Level Distribution":
                cursor.execute('''
                    SELECT risk_level, COUNT(*) as count FROM sim_requests GROUP BY risk_level
                ''')
                headers = ["Risk Level", "Total Count"]
            else: # Alert Summary
                cursor.execute('''
                    SELECT a.alert_id, c.name, c.mobile_number, a.message, a.risk_score, a.timestamp
                    FROM alerts a
                    JOIN sim_requests sr ON a.request_id = sr.request_id
                    JOIN customers c ON sr.customer_id = c.customer_id
                ''')
                headers = ["Alert ID", "Name", "Mobile Number", "Message", "Risk Score", "Timestamp"]

            data = cursor.fetchall()
            return headers, data


# ============================================================================
# FRAUD DETECTION ENGINE LOGIC
# ============================================================================
class FraudDetectionEngine:
    """Calculates risk score and risk classification based on input parameters."""
    
    @staticmethod
    def calculate_risk(device_change, multiple_requests, new_location, frequent_changes):
        score = 0
        if device_change:
            score += 20
        if multiple_requests:
            score += 30
        if new_location:
            score += 25
        if frequent_changes:
            score += 25

        score = min(score, 100)

        if score <= 25:
            level = "Low Risk"
        elif score <= 50:
            level = "Medium Risk"
        elif score <= 75:
            level = "High Risk"
        else:
            level = "Critical"

        return score, level


# ============================================================================
# MAIN APPLICATION GUI CLASS
# ============================================================================
class SIMSwapApp(tk.Tk):
    """Main Application Window with Sidebar, 12 Themes + Custom Theme Creator, and Dynamic Frames."""

    # 12 Default Preset Themes
    THEMES = {
        "Light Default": {
            "sidebar": "#1e293b", "sidebar_btn": "#334155", "sidebar_active": "#2563eb",
            "bg": "#f8fafc", "card_bg": "#ffffff", "text": "#0f172a", "text_muted": "#64748b",
            "input_bg": "#ffffff", "border": "#e2e8f0", "low_risk": "#16a34a",
            "med_risk": "#d97706", "high_risk": "#ea580c", "critical_risk": "#dc2626"
        },
        "Dark Modern": {
            "sidebar": "#0f172a", "sidebar_btn": "#1e293b", "sidebar_active": "#3b82f6",
            "bg": "#090d16", "card_bg": "#1e293b", "text": "#f8fafc", "text_muted": "#94a3b8",
            "input_bg": "#0f172a", "border": "#334155", "low_risk": "#22c55e",
            "med_risk": "#f59e0b", "high_risk": "#f97316", "critical_risk": "#ef4444"
        },
        "Solarized Light": {
            "sidebar": "#073642", "sidebar_btn": "#586e75", "sidebar_active": "#268bd2",
            "bg": "#fdf6e3", "card_bg": "#eee8d5", "text": "#073642", "text_muted": "#657b83",
            "input_bg": "#fdf6e3", "border": "#93a1a1", "low_risk": "#859900",
            "med_risk": "#b58900", "high_risk": "#cb4b16", "critical_risk": "#dc322f"
        },
        "Solarized Dark": {
            "sidebar": "#002b36", "sidebar_btn": "#073642", "sidebar_active": "#2b6cb0",
            "bg": "#002b36", "card_bg": "#073642", "text": "#93a1a1", "text_muted": "#657b83",
            "input_bg": "#002b36", "border": "#586e75", "low_risk": "#859900",
            "med_risk": "#b58900", "high_risk": "#cb4b16", "critical_risk": "#dc322f"
        },
        "Nord": {
            "sidebar": "#2e3440", "sidebar_btn": "#3b4252", "sidebar_active": "#88c0d0",
            "bg": "#434c5e", "card_bg": "#2e3440", "text": "#eceff4", "text_muted": "#d8dee9",
            "input_bg": "#3b4252", "border": "#4c566a", "low_risk": "#a3be8c",
            "med_risk": "#ebcb8b", "high_risk": "#d08770", "critical_risk": "#bf616a"
        },
        "Cyberpunk / Neon": {
            "sidebar": "#120428", "sidebar_btn": "#240046", "sidebar_active": "#ff007f",
            "bg": "#0a0017", "card_bg": "#1d0036", "text": "#00f5d4", "text_muted": "#7b2cbf",
            "input_bg": "#120428", "border": "#ff007f", "low_risk": "#00f5d4",
            "med_risk": "#fee440", "high_risk": "#ff70a6", "critical_risk": "#ff007f"
        },
        "Forest Greens": {
            "sidebar": "#143601", "sidebar_btn": "#1a4301", "sidebar_active": "#538d22",
            "bg": "#f4f9f4", "card_bg": "#e8f5e9", "text": "#0e2700", "text_muted": "#245501",
            "input_bg": "#ffffff", "border": "#c8e6c9", "low_risk": "#2d6a4f",
            "med_risk": "#b58900", "high_risk": "#d97706", "critical_risk": "#990000"
        },
        "Emerald Luxury": {
            "sidebar": "#064e3b", "sidebar_btn": "#047857", "sidebar_active": "#10b981",
            "bg": "#022c22", "card_bg": "#064e3b", "text": "#ecfdf5", "text_muted": "#6ee7b7",
            "input_bg": "#022c22", "border": "#047857", "low_risk": "#34d399",
            "med_risk": "#fbbf24", "high_risk": "#f97316", "critical_risk": "#f87171"
        },
        "Ocean / Dracula": {
            "sidebar": "#282a36", "sidebar_btn": "#44475a", "sidebar_active": "#bd93f9",
            "bg": "#1e1f29", "card_bg": "#282a36", "text": "#f8f8f2", "text_muted": "#6272a4",
            "input_bg": "#44475a", "border": "#6272a4", "low_risk": "#50fa7b",
            "med_risk": "#f1fa8c", "high_risk": "#ffb86c", "critical_risk": "#ff5555"
        },
        "Amber Glow": {
            "sidebar": "#291d09", "sidebar_btn": "#453216", "sidebar_active": "#d97706",
            "bg": "#120c04", "card_bg": "#291d09", "text": "#fef3c7", "text_muted": "#b45309",
            "input_bg": "#1c1307", "border": "#78350f", "low_risk": "#10b981",
            "med_risk": "#f59e0b", "high_risk": "#f97316", "critical_risk": "#ef4444"
        },
        "Soft Pastel": {
            "sidebar": "#4a5568", "sidebar_btn": "#718096", "sidebar_active": "#b8c2cc",
            "bg": "#f7fafc", "card_bg": "#edf2f7", "text": "#2d3748", "text_muted": "#718096",
            "input_bg": "#ffffff", "border": "#cbd5e0", "low_risk": "#48bb78",
            "med_risk": "#ed8936", "high_risk": "#dd6b20", "critical_risk": "#e53e3e"
        },
        "High Contrast": {
            "sidebar": "#000000", "sidebar_btn": "#1a1a1a", "sidebar_active": "#ffff00",
            "bg": "#000000", "card_bg": "#111111", "text": "#ffffff", "text_muted": "#cccccc",
            "input_bg": "#000000", "border": "#ffffff", "low_risk": "#00ff00",
            "med_risk": "#ffff00", "high_risk": "#ffaa00", "critical_risk": "#ff0000"
        }
    }

    def __init__(self):
        super().__init__()
        self.title("SIM Swap Fraud Detection System")
        self.geometry("1100x680")
        self.minsize(950, 600)

        # Active Theme State
        self.current_theme_name = "Light Default"
        self.colors = dict(self.THEMES[self.current_theme_name])

        self.db = DatabaseManager()

        self.grid_columnconfigure(1, weight=1)
        self.grid_rowconfigure(0, weight=1)

        self.current_screen_key = "dashboard"

        self.create_sidebar()
        self.create_content_area()

        # Apply initial theme & show dashboard
        self.apply_theme()
        self.show_screen("dashboard")

    def change_preset_theme(self, event=None):
        """Switches to selected preset theme from combobox."""
        selected_theme = self.cmb_theme_select.get()
        if selected_theme in self.THEMES:
            self.current_theme_name = selected_theme
            self.colors = dict(self.THEMES[selected_theme])
            self.apply_theme()
            self.show_screen(self.current_screen_key)

    def open_custom_theme_dialog(self):
        """Opens modal dialog allowing users to pick custom UI colors."""
        dialog = tk.Toplevel(self)
        dialog.title("Custom Theme Creator")
        dialog.geometry("450x520")
        dialog.transient(self)
        dialog.grab_set()
        dialog.configure(bg=self.colors["bg"])

        lbl = tk.Label(dialog, text="Customize Color Palette", font=("Segoe UI", 12, "bold"), bg=self.colors["bg"], fg=self.colors["text"])
        lbl.pack(pady=10)

        temp_colors = dict(self.colors)

        color_items = [
            ("Main Background", "bg"),
            ("Card Background", "card_bg"),
            ("Sidebar Background", "sidebar"),
            ("Sidebar Button", "sidebar_btn"),
            ("Active Highlight", "sidebar_active"),
            ("Text Color", "text"),
            ("Low Risk Color", "low_risk"),
            ("Critical Risk Color", "critical_risk")
        ]

        def pick_color(key, btn_preview):
            color = colorchooser.askcolor(initialcolor=temp_colors[key], title=f"Choose {key}")[1]
            if color:
                temp_colors[key] = color
                btn_preview.configure(bg=color)

        frame_opts = tk.Frame(dialog, bg=self.colors["bg"])
        frame_opts.pack(fill="both", expand=True, padx=20, pady=5)

        for name, key in color_items:
            row = tk.Frame(frame_opts, bg=self.colors["bg"])
            row.pack(fill="x", pady=4)
            
            tk.Label(row, text=name, font=("Segoe UI", 9, "bold"), width=20, anchor="w", bg=self.colors["bg"], fg=self.colors["text"]).pack(side="left")
            btn_prev = tk.Button(row, bg=temp_colors[key], width=8, bd=1, relief="solid")
            btn_prev.pack(side="right")
            btn_prev.configure(command=lambda k=key, b=btn_prev: pick_color(k, b))

        def apply_custom():
            self.THEMES["Custom User Theme"] = temp_colors
            self.current_theme_name = "Custom User Theme"
            self.colors = dict(temp_colors)
            
            # Refresh Combobox values
            self.cmb_theme_select["values"] = list(self.THEMES.keys())
            self.cmb_theme_select.set("Custom User Theme")
            
            self.apply_theme()
            self.show_screen(self.current_screen_key)
            dialog.destroy()

        tk.Button(dialog, text="Apply Custom Theme", font=("Segoe UI", 10, "bold"), bg="#16a34a", fg="white", pady=6, bd=0, command=apply_custom).pack(fill="x", padx=20, pady=15)

    def apply_theme(self):
        """Configures global window colors and Treeview styles based on active theme."""
        self.configure(bg=self.colors["bg"])
        
        # Configure TTK Styles for Tables and Controls
        style = ttk.Style()
        style.theme_use("clam")
        
        style.configure(
            "Treeview",
            background=self.colors["card_bg"],
            foreground=self.colors["text"],
            fieldbackground=self.colors["card_bg"],
            rowheight=26,
            bordercolor=self.colors["border"],
            borderwidth=1
        )
        style.configure(
            "Treeview.Heading",
            background=self.colors["sidebar_btn"],
            foreground="#ffffff",
            font=("Segoe UI", 9, "bold")
        )
        style.map("Treeview", background=[('selected', self.colors["sidebar_active"])])

    def create_sidebar(self):
        """Creates the permanent sidebar navigation panel with theme management."""
        self.sidebar = tk.Frame(self, bg=self.colors["sidebar"], width=230)
        self.sidebar.grid(row=0, column=0, sticky="nsew")
        self.sidebar.grid_propagate(False)

        lbl_title = tk.Label(
            self.sidebar, 
            text="SIM SWAP\nDETECTOR", 
            font=("Segoe UI", 14, "bold"), 
            bg=self.colors["sidebar"], 
            fg="#ffffff",
            justify="center"
        )
        lbl_title.pack(pady=15, padx=10)

        nav_items = [
            ("Dashboard", "dashboard"),
            ("New SIM Swap Request", "new_request"),
            ("Alert Management", "alerts"),
            ("Fraud Incidents", "incidents"),
            ("Database Records", "records"),
            ("Reports", "reports")
        ]

        self.nav_buttons = {}
        for text, key in nav_items:
            btn = tk.Button(
                self.sidebar,
                text=text,
                font=("Segoe UI", 10, "bold"),
                bg=self.colors["sidebar_btn"],
                fg="#ffffff",
                activebackground=self.colors["sidebar_active"],
                activeforeground="#ffffff",
                bd=0,
                pady=9,
                anchor="w",
                padx=15,
                cursor="hand2",
                command=lambda k=key: self.show_screen(k)
            )
            btn.pack(fill="x", padx=10, pady=2)
            self.nav_buttons[key] = btn

        # Theme Section Frame at Sidebar Bottom
        theme_frame = tk.Frame(self.sidebar, bg=self.colors["sidebar"])
        theme_frame.pack(side="bottom", fill="x", padx=10, pady=15)

        tk.Label(theme_frame, text="Appearance Theme:", font=("Segoe UI", 8, "bold"), bg=self.colors["sidebar"], fg="#ffffff").pack(anchor="w", pady=(0, 2))

        self.cmb_theme_select = ttk.Combobox(
            theme_frame,
            values=list(self.THEMES.keys()),
            state="readonly",
            font=("Segoe UI", 9)
        )
        self.cmb_theme_select.set(self.current_theme_name)
        self.cmb_theme_select.pack(fill="x", pady=(0, 5))
        self.cmb_theme_select.bind("<<ComboboxSelected>>", self.change_preset_theme)

        btn_custom = tk.Button(
            theme_frame,
            text="+ Custom Theme",
            font=("Segoe UI", 8, "bold"),
            bg="#0284c7",
            fg="#ffffff",
            activebackground="#0369a1",
            activeforeground="#ffffff",
            bd=0,
            pady=4,
            cursor="hand2",
            command=self.open_custom_theme_dialog
        )
        btn_custom.pack(fill="x")

    def create_content_area(self):
        """Creates the main container frame for dynamic view switching."""
        self.content_frame = tk.Frame(self, bg=self.colors["bg"])
        self.content_frame.grid(row=0, column=1, sticky="nsew", padx=20, pady=20)
        self.content_frame.grid_columnconfigure(0, weight=1)
        self.content_frame.grid_rowconfigure(0, weight=1)

        self.screens = {}

    def show_screen(self, key):
        """Switches the active visible screen."""
        self.current_screen_key = key

        self.sidebar.configure(bg=self.colors["sidebar"])

        for k, btn in self.nav_buttons.items():
            if k == key:
                btn.configure(bg=self.colors["sidebar_active"])
            else:
                btn.configure(bg=self.colors["sidebar_btn"])

        if key in self.screens:
            self.screens[key].destroy()

        if key == "dashboard":
            frame = self.build_dashboard()
        elif key == "new_request":
            frame = self.build_new_request()
        elif key == "alerts":
            frame = self.build_alerts()
        elif key == "incidents":
            frame = self.build_incidents()
        elif key == "records":
            frame = self.build_records()
        elif key == "reports":
            frame = self.build_reports()

        self.screens[key] = frame
        frame.grid(row=0, column=0, sticky="nsew")

    # =========================================================================
    # 1. DASHBOARD SCREEN
    # =========================================================================
    def build_dashboard(self):
        frame = tk.Frame(self.content_frame, bg=self.colors["bg"])
        frame.grid_columnconfigure((0, 1, 2, 3), weight=1)

        lbl_head = tk.Label(frame, text="System Overview Dashboard", font=("Segoe UI", 18, "bold"), bg=self.colors["bg"], fg=self.colors["text"])
        lbl_head.grid(row=0, column=0, columnspan=4, sticky="w", pady=(0, 15))

        stats = self.db.get_dashboard_stats()

        cards = [
            ("Total Requests", stats["total"], "#2563eb"),
            ("Safe Requests", stats["safe"], self.colors["low_risk"]),
            ("Medium Risk", stats["medium"], self.colors["med_risk"]),
            ("High/Critical Risk", stats["high"], self.colors["critical_risk"]),
        ]

        for idx, (title, count, color) in enumerate(cards):
            card = tk.Frame(frame, bg=self.colors["card_bg"], highlightbackground=self.colors["border"], highlightthickness=1)
            card.grid(row=1, column=idx, sticky="nsew", padx=5, pady=5)
            
            strip = tk.Frame(card, bg=color, width=6)
            strip.pack(side="left", fill="y")

            content = tk.Frame(card, bg=self.colors["card_bg"], padx=10, pady=10)
            content.pack(side="left", fill="both", expand=True)

            tk.Label(content, text=title, font=("Segoe UI", 9, "bold"), fg=self.colors["text_muted"], bg=self.colors["card_bg"]).pack(anchor="w")
            tk.Label(content, text=str(count), font=("Segoe UI", 18, "bold"), fg=self.colors["text"], bg=self.colors["card_bg"]).pack(anchor="w")

        lbl_table = tk.Label(frame, text="Recent Fraud Alerts", font=("Segoe UI", 13, "bold"), bg=self.colors["bg"], fg=self.colors["text"])
        lbl_table.grid(row=2, column=0, columnspan=4, sticky="w", pady=(20, 10))

        tree_frame = tk.Frame(frame, bg=self.colors["card_bg"])
        tree_frame.grid(row=3, column=0, columnspan=4, sticky="nsew")
        frame.grid_rowconfigure(3, weight=1)

        cols = ("Alert ID", "Customer Name", "Mobile Number", "Risk Score", "Risk Level", "Status", "Timestamp")
        tree = ttk.Treeview(tree_frame, columns=cols, show="headings", height=10)
        
        for col in cols:
            tree.heading(col, text=col)
            tree.column(col, anchor="center", width=120)

        vsb = ttk.Scrollbar(tree_frame, orient="vertical", command=tree.yview)
        tree.configure(yscrollcommand=vsb.set)

        tree.pack(side="left", fill="both", expand=True)
        vsb.pack(side="right", fill="y")

        tree.tag_configure("Low Risk", foreground=self.colors["low_risk"])
        tree.tag_configure("Medium Risk", foreground=self.colors["med_risk"])
        tree.tag_configure("High Risk", foreground=self.colors["high_risk"])
        tree.tag_configure("Critical", foreground=self.colors["critical_risk"])

        alerts = self.db.get_recent_alerts()
        for row in alerts:
            tree.insert("", "end", values=row, tags=(row[4],))

        return frame

    # =========================================================================
    # 2. NEW SIM SWAP REQUEST SCREEN
    # =========================================================================
    def build_new_request(self):
        frame = tk.Frame(self.content_frame, bg=self.colors["bg"])

        lbl_head = tk.Label(frame, text="New SIM Swap Request Entry", font=("Segoe UI", 18, "bold"), bg=self.colors["bg"], fg=self.colors["text"])
        lbl_head.pack(anchor="w", pady=(0, 15))

        form_card = tk.Frame(frame, bg=self.colors["card_bg"], padx=20, pady=20, highlightbackground=self.colors["border"], highlightthickness=1)
        form_card.pack(fill="both", expand=True)

        fields_frame = tk.Frame(form_card, bg=self.colors["card_bg"])
        fields_frame.pack(fill="x")

        labels = [
            "Customer Name", "Mobile Number", "Aadhaar Number",
            "Previous SIM Issue Date (YYYY-MM-DD)", "Current Request Date (YYYY-MM-DD)",
            "Location", "Reason for SIM Replacement"
        ]

        self.form_entries = {}
        for idx, lbl_text in enumerate(labels):
            row = idx // 2
            col = (idx % 2) * 2

            lbl = tk.Label(fields_frame, text=lbl_text, font=("Segoe UI", 9, "bold"), bg=self.colors["card_bg"], fg=self.colors["text"])
            lbl.grid(row=row*2, column=col, sticky="w", padx=10, pady=(5, 2))

            entry = tk.Entry(fields_frame, font=("Segoe UI", 10), bd=1, relief="solid", bg=self.colors["input_bg"], fg=self.colors["text"], insertbackground=self.colors["text"])
            entry.grid(row=row*2+1, column=col, sticky="ew", padx=10, pady=(0, 10))
            self.form_entries[lbl_text] = entry

        today_str = datetime.now().strftime("%Y-%m-%d")
        self.form_entries["Current Request Date (YYYY-MM-DD)"].insert(0, today_str)
        self.form_entries["Previous SIM Issue Date (YYYY-MM-DD)"].insert(0, "2023-01-15")

        fields_frame.grid_columnconfigure((0, 2), weight=1)

        chk_frame = tk.LabelFrame(form_card, text="Risk Indicators", font=("Segoe UI", 10, "bold"), bg=self.colors["card_bg"], fg=self.colors["text"], padx=15, pady=10)
        chk_frame.pack(fill="x", pady=15)

        self.var_device = tk.BooleanVar()
        self.var_multiple = tk.BooleanVar()
        self.var_location = tk.BooleanVar()
        self.var_frequent = tk.BooleanVar()

        chk_opts = {"bg": self.colors["card_bg"], "fg": self.colors["text"], "activebackground": self.colors["card_bg"], "activeforeground": self.colors["text"], "selectcolor": self.colors["input_bg"]}

        tk.Checkbutton(chk_frame, text="Recent Device Change (+20 Score)", variable=self.var_device, font=("Segoe UI", 9), **chk_opts).grid(row=0, column=0, sticky="w", padx=10, pady=5)
        tk.Checkbutton(chk_frame, text="Multiple SIM Requests (+30 Score)", variable=self.var_multiple, font=("Segoe UI", 9), **chk_opts).grid(row=0, column=1, sticky="w", padx=10, pady=5)
        tk.Checkbutton(chk_frame, text="New Location Request (+25 Score)", variable=self.var_location, font=("Segoe UI", 9), **chk_opts).grid(row=1, column=0, sticky="w", padx=10, pady=5)
        tk.Checkbutton(chk_frame, text="Frequent Number Changes (+25 Score)", variable=self.var_frequent, font=("Segoe UI", 9), **chk_opts).grid(row=1, column=1, sticky="w", padx=10, pady=5)

        self.lbl_score_display = tk.Label(form_card, text="Calculated Risk Score: 0 | Risk Level: Low Risk", font=("Segoe UI", 11, "bold"), bg=self.colors["sidebar_btn"], fg="#ffffff", pady=8)
        self.lbl_score_display.pack(fill="x", pady=10)

        btn_frame = tk.Frame(form_card, bg=self.colors["card_bg"])
        btn_frame.pack(fill="x", pady=10)

        tk.Button(btn_frame, text="Analyze Risk", font=("Segoe UI", 10, "bold"), bg="#3b82f6", fg="white", padx=15, pady=6, bd=0, cursor="hand2", command=self.on_analyze_risk).pack(side="left", padx=5)
        tk.Button(btn_frame, text="Save Request", font=("Segoe UI", 10, "bold"), bg="#16a34a", fg="white", padx=15, pady=6, bd=0, cursor="hand2", command=self.on_save_request).pack(side="left", padx=5)
        tk.Button(btn_frame, text="Clear Form", font=("Segoe UI", 10, "bold"), bg="#64748b", fg="white", padx=15, pady=6, bd=0, cursor="hand2", command=self.on_clear_form).pack(side="left", padx=5)

        return frame

    def on_analyze_risk(self):
        score, level = FraudDetectionEngine.calculate_risk(
            self.var_device.get(),
            self.var_multiple.get(),
            self.var_location.get(),
            self.var_frequent.get()
        )
        self.lbl_score_display.config(text=f"Calculated Risk Score: {score} | Risk Level: {level}")
        return score, level

    def on_save_request(self):
        name = self.form_entries["Customer Name"].get().strip()
        mobile = self.form_entries["Mobile Number"].get().strip()
        aadhaar = self.form_entries["Aadhaar Number"].get().strip()
        prev_date = self.form_entries["Previous SIM Issue Date (YYYY-MM-DD)"].get().strip()
        curr_date = self.form_entries["Current Request Date (YYYY-MM-DD)"].get().strip()
        location = self.form_entries["Location"].get().strip()
        reason = self.form_entries["Reason for SIM Replacement"].get().strip()

        if not name or not mobile or not aadhaar or not location or not reason:
            messagebox.showwarning("Validation Error", "Please fill in all input fields.")
            return

        score, level = self.on_analyze_risk()

        self.db.insert_sim_swap_transaction(name, mobile, aadhaar, prev_date, curr_date, location, reason, score, level)

        if score > 50:
            messagebox.showwarning("Alert Generated", f"SIM Request recorded successfully!\n\nFRAUD ALERT GENERATED:\nRisk Score: {score}\nRisk Level: {level}")
        else:
            messagebox.showinfo("Success", "SIM Swap request recorded successfully. Low risk level.")

        self.on_clear_form()

    def on_clear_form(self):
        for entry in self.form_entries.values():
            entry.delete(0, tk.END)
        self.var_device.set(False)
        self.var_multiple.set(False)
        self.var_location.set(False)
        self.var_frequent.set(False)
        self.form_entries["Current Request Date (YYYY-MM-DD)"].insert(0, datetime.now().strftime("%Y-%m-%d"))
        self.form_entries["Previous SIM Issue Date (YYYY-MM-DD)"].insert(0, "2023-01-15")
        self.lbl_score_display.config(text="Calculated Risk Score: 0 | Risk Level: Low Risk")

    # =========================================================================
    # 3. ALERT MANAGEMENT SCREEN
    # =========================================================================
    def build_alerts(self):
        frame = tk.Frame(self.content_frame, bg=self.colors["bg"])

        lbl_head = tk.Label(frame, text="Alert Management", font=("Segoe UI", 18, "bold"), bg=self.colors["bg"], fg=self.colors["text"])
        lbl_head.pack(anchor="w", pady=(0, 15))

        tree_frame = tk.Frame(frame, bg=self.colors["card_bg"])
        tree_frame.pack(fill="both", expand=True)

        cols = ("Alert ID", "Customer Name", "Mobile Number", "Risk Score", "Risk Level", "Status", "Timestamp")
        tree = ttk.Treeview(tree_frame, columns=cols, show="headings")

        for col in cols:
            tree.heading(col, text=col)
            tree.column(col, anchor="center")

        vsb = ttk.Scrollbar(tree_frame, orient="vertical", command=tree.yview)
        tree.configure(yscrollcommand=vsb.set)

        tree.pack(side="left", fill="both", expand=True)
        vsb.pack(side="right", fill="y")

        alerts = self.db.get_recent_alerts(limit=100)
        for row in alerts:
            tree.insert("", "end", values=row)

        return frame

    # =========================================================================
    # 4. FRAUD INCIDENTS SCREEN
    # =========================================================================
    def build_incidents(self):
        frame = tk.Frame(self.content_frame, bg=self.colors["bg"])

        lbl_head = tk.Label(frame, text="Fraud Incidents Management", font=("Segoe UI", 18, "bold"), bg=self.colors["bg"], fg=self.colors["text"])
        lbl_head.pack(anchor="w", pady=(0, 15))

        tree_frame = tk.Frame(frame, bg=self.colors["card_bg"])
        tree_frame.pack(fill="both", expand=True)

        cols = ("Incident ID", "Customer Name", "Risk Score", "Risk Level", "Status", "Created At")
        self.incidents_tree = ttk.Treeview(tree_frame, columns=cols, show="headings")

        for col in cols:
            self.incidents_tree.heading(col, text=col)
            self.incidents_tree.column(col, anchor="center")

        vsb = ttk.Scrollbar(tree_frame, orient="vertical", command=self.incidents_tree.yview)
        self.incidents_tree.configure(yscrollcommand=vsb.set)

        self.incidents_tree.pack(side="left", fill="both", expand=True)
        vsb.pack(side="right", fill="y")

        btn_frame = tk.Frame(frame, bg=self.colors["bg"])
        btn_frame.pack(fill="x", pady=15)

        tk.Button(btn_frame, text="Mark as Investigating", font=("Segoe UI", 9, "bold"), bg="#eab308", fg="white", padx=10, pady=5, bd=0, cursor="hand2", command=lambda: self.update_incident("Investigating")).pack(side="left", padx=5)
        tk.Button(btn_frame, text="Mark as Resolved", font=("Segoe UI", 9, "bold"), bg="#16a34a", fg="white", padx=10, pady=5, bd=0, cursor="hand2", command=lambda: self.update_incident("Resolved")).pack(side="left", padx=5)
        tk.Button(btn_frame, text="Delete Record", font=("Segoe UI", 9, "bold"), bg="#dc2626", fg="white", padx=10, pady=5, bd=0, cursor="hand2", command=self.delete_selected_incident).pack(side="left", padx=5)

        self.refresh_incidents_list()
        return frame

    def refresh_incidents_list(self):
        for item in self.incidents_tree.get_children():
            self.incidents_tree.delete(item)

        records = self.db.get_all_incidents()
        for row in records:
            self.incidents_tree.insert("", "end", values=row)

    def update_incident(self, new_status):
        selected = self.incidents_tree.selection()
        if not selected:
            messagebox.showwarning("Selection Error", "Please select an incident from the table.")
            return

        incident_id = self.incidents_tree.item(selected[0])["values"][0]
        self.db.update_incident_status(incident_id, new_status)
        messagebox.showinfo("Updated", f"Incident #{incident_id} marked as '{new_status}'.")
        self.refresh_incidents_list()

    def delete_selected_incident(self):
        selected = self.incidents_tree.selection()
        if not selected:
            messagebox.showwarning("Selection Error", "Please select an incident from the table.")
            return

        incident_id = self.incidents_tree.item(selected[0])["values"][0]
        if messagebox.askyesno("Confirm Delete", f"Are you sure you want to delete Incident #{incident_id}?"):
            self.db.delete_incident(incident_id)
            messagebox.showinfo("Deleted", "Record deleted successfully.")
            self.refresh_incidents_list()

    # =========================================================================
    # 5. DATABASE RECORDS SCREEN
    # =========================================================================
    def build_records(self):
        frame = tk.Frame(self.content_frame, bg=self.colors["bg"])

        lbl_head = tk.Label(frame, text="Database Records Explorer", font=("Segoe UI", 18, "bold"), bg=self.colors["bg"], fg=self.colors["text"])
        lbl_head.pack(anchor="w", pady=(0, 15))

        control_frame = tk.Frame(frame, bg=self.colors["card_bg"], padx=10, pady=10, highlightbackground=self.colors["border"], highlightthickness=1)
        control_frame.pack(fill="x", pady=(0, 10))

        tk.Label(control_frame, text="Search (Name/Mobile):", font=("Segoe UI", 9, "bold"), bg=self.colors["card_bg"], fg=self.colors["text"]).pack(side="left", padx=5)
        self.ent_search = tk.Entry(control_frame, font=("Segoe UI", 10), width=20, bg=self.colors["input_bg"], fg=self.colors["text"], insertbackground=self.colors["text"])
        self.ent_search.pack(side="left", padx=5)

        tk.Label(control_frame, text="Filter Risk:", font=("Segoe UI", 9, "bold"), bg=self.colors["card_bg"], fg=self.colors["text"]).pack(side="left", padx=(15, 5))
        self.cmb_risk = ttk.Combobox(control_frame, values=["All", "Low Risk", "Medium Risk", "High Risk", "Critical"], state="readonly", width=12)
        self.cmb_risk.set("All")
        self.cmb_risk.pack(side="left", padx=5)

        tk.Button(control_frame, text="Apply Filter", font=("Segoe UI", 9, "bold"), bg="#2563eb", fg="white", padx=10, bd=0, cursor="hand2", command=self.refresh_records_list).pack(side="left", padx=10)
        tk.Button(control_frame, text="Refresh", font=("Segoe UI", 9, "bold"), bg="#64748b", fg="white", padx=10, bd=0, cursor="hand2", command=self.refresh_records_list).pack(side="left", padx=5)

        tree_frame = tk.Frame(frame, bg=self.colors["card_bg"])
        tree_frame.pack(fill="both", expand=True)

        cols = ("Req ID", "Name", "Mobile Number", "Aadhaar", "Location", "Reason", "Score", "Risk Level", "Date")
        self.records_tree = ttk.Treeview(tree_frame, columns=cols, show="headings")

        for col in cols:
            self.records_tree.heading(col, text=col)
            self.records_tree.column(col, anchor="center", width=100)

        vsb = ttk.Scrollbar(tree_frame, orient="vertical", command=self.records_tree.yview)
        self.records_tree.configure(yscrollcommand=vsb.set)

        self.records_tree.pack(side="left", fill="both", expand=True)
        vsb.pack(side="right", fill="y")

        self.refresh_records_list()
        return frame

    def refresh_records_list(self):
        for item in self.records_tree.get_children():
            self.records_tree.delete(item)

        search_term = self.ent_search.get().strip()
        risk_filter = self.cmb_risk.get()

        data = self.db.get_database_records(search_term, risk_filter)
        for row in data:
            self.records_tree.insert("", "end", values=row)

    # =========================================================================
    # 6. REPORTS SCREEN
    # =========================================================================
    def build_reports(self):
        frame = tk.Frame(self.content_frame, bg=self.colors["bg"])

        lbl_head = tk.Label(frame, text="System Reports & Analytics", font=("Segoe UI", 18, "bold"), bg=self.colors["bg"], fg=self.colors["text"])
        lbl_head.pack(anchor="w", pady=(0, 15))

        bar = tk.Frame(frame, bg=self.colors["card_bg"], padx=10, pady=10, highlightbackground=self.colors["border"], highlightthickness=1)
        bar.pack(fill="x", pady=(0, 10))

        tk.Label(bar, text="Select Report Type:", font=("Segoe UI", 10, "bold"), bg=self.colors["card_bg"], fg=self.colors["text"]).pack(side="left", padx=5)
        
        self.cmb_report = ttk.Combobox(
            bar, 
            values=["Total Requests", "Fraud Incidents", "Risk Level Distribution", "Alert Summary"], 
            state="readonly", 
            width=22
        )
        self.cmb_report.set("Total Requests")
        self.cmb_report.pack(side="left", padx=5)

        tk.Button(bar, text="Generate Report", font=("Segoe UI", 9, "bold"), bg="#2563eb", fg="white", padx=10, bd=0, cursor="hand2", command=self.generate_report).pack(side="left", padx=10)
        tk.Button(bar, text="Export to CSV", font=("Segoe UI", 9, "bold"), bg="#16a34a", fg="white", padx=10, bd=0, cursor="hand2", command=self.export_csv).pack(side="right", padx=5)

        self.report_tree_frame = tk.Frame(frame, bg=self.colors["card_bg"])
        self.report_tree_frame.pack(fill="both", expand=True)

        self.generate_report()
        return frame

    def generate_report(self):
        for widget in self.report_tree_frame.winfo_children():
            widget.destroy()

        report_type = self.cmb_report.get()
        headers, data = self.db.get_report_data(report_type)

        self.current_report_headers = headers
        self.current_report_data = data

        tree = ttk.Treeview(self.report_tree_frame, columns=headers, show="headings")
        for col in headers:
            tree.heading(col, text=col)
            tree.column(col, anchor="center")

        vsb = ttk.Scrollbar(self.report_tree_frame, orient="vertical", command=tree.yview)
        tree.configure(yscrollcommand=vsb.set)

        tree.pack(side="left", fill="both", expand=True)
        vsb.pack(side="right", fill="y")

        for row in data:
            tree.insert("", "end", values=row)

    def export_csv(self):
        if not hasattr(self, 'current_report_data') or not self.current_report_data:
            messagebox.showwarning("Export Warning", "No report data available to export.")
            return

        filename = filedialog.asksaveasfilename(
            defaultextension=".csv",
            filetypes=[("CSV Files", "*.csv"), ("All Files", "*.*")],
            title="Save Report As"
        )

        if filename:
            try:
                with open(filename, mode="w", newline="", encoding="utf-8") as f:
                    writer = csv.writer(f)
                    writer.writerow(self.current_report_headers)
                    writer.writerows(self.current_report_data)
                messagebox.showinfo("Export Successful", f"Report successfully saved to:\n{filename}")
            except Exception as e:
                messagebox.showerror("Export Error", f"Failed to export CSV: {str(e)}")


# ============================================================================
# ENTRY POINT
# ============================================================================
if __name__ == "__main__":
    app = SIMSwapApp()
    app.mainloop()
