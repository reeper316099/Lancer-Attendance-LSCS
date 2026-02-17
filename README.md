# Lancer Attendance (LSCS)

A Flask + SQLite attendance system for La Salle College using RFID/NFC cards. The project includes:
- a live public check-in display,
- an authenticated admin dashboard,
- user/card registration,
- attendance reporting,
- optional Raspberry Pi RFID scanner integration.

## Table of Contents
- [Project Overview](#project-overview)
- [Core Features](#core-features)
- [Tech Stack](#tech-stack)
- [Repository Structure](#repository-structure)
- [How the System Works](#how-the-system-works)
- [Local Development Setup](#local-development-setup)
- [Running the Application](#running-the-application)
- [Configuration](#configuration)
- [Database Schema](#database-schema)
- [API Reference](#api-reference)
- [Deployment Notes (Raspberry Pi)](#deployment-notes-raspberry-pi)
- [Troubleshooting](#troubleshooting)

## Project Overview
This system tracks attendance by scanning RFID/NFC cards. Each scan toggles a user between **checked in** and **checked out**. Admin users can also manage users manually, view reports, and configure settings such as max occupancy and automatic end-of-day checkout.

The web server and scanner are separated:
- `app.py` handles web UI, APIs, authentication, and real-time updates via WebSockets.
- `rfid_scanner.py` interfaces with PN532 hardware (or simulation mode if unavailable).
- `models.py` provides all database operations.

## Core Features
- **Admin authentication** with session-protected routes.
- **First-run admin setup** at `/setup-admin` (disabled once admins exist).
- **RFID card registration** and user management (create/update/delete users).
- **Automatic check-in/check-out flow** when cards are scanned.
- **Manual check-in/check-out** controls from admin tools.
- **Live display page** (`/display`) that updates with check-in events.
- **Reporting endpoints** for daily, weekly, and per-user attendance history.
- **System settings** management via API:
  - `max_occupancy`
  - `auto_checkout_time`
  - `auto_checkout_enabled`
- **Background auto-checkout scheduler** in the web server.

## Tech Stack
- **Backend:** Python, Flask, Flask-SocketIO
- **Database:** SQLite
- **Frontend:** Jinja templates + static JS/CSS
- **Hardware integration (optional):** Adafruit PN532 (I2C)

## Repository Structure
```text
.
├── app.py                    # Flask app + routes + Socket.IO + scheduler
├── models.py                 # SQLite schema + query/update helpers
├── rfid_scanner.py           # PN532 scanner loop + scan processing
├── config.py                 # Runtime configuration values
├── start.sh                  # Quick-start script
├── setup.sh                  # Raspberry Pi setup helper script
├── templates/                # HTML templates
├── static/                   # Frontend assets (CSS, JS, images)
├── requirements.txt          # Python dependencies
└── README.md
```

## How the System Works
1. A card is scanned by `rfid_scanner.py`.
2. The scanner resolves the card UID to a user in the `users` table.
3. If the user is currently out, a check-in record is created.
4. If the user is currently in, their open record is checked out.
5. The frontend can request current status via API and receive real-time events via Socket.IO.

## Local Development Setup
> These steps are for local/dev environments. Hardware is optional.

1. Create and activate a virtual environment:
   ```bash
   python3 -m venv venv
   source venv/bin/activate
   ```

2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

3. Initialize the database:
   ```bash
   python3 models.py
   ```

## Running the Application
### Option A: Quick start script
```bash
bash start.sh
```

### Option B: Run manually
Terminal 1 (web server):
```bash
python3 app.py
```

Terminal 2 (RFID scanner, optional):
```bash
python3 rfid_scanner.py
```

### First-run setup
- Open `http://localhost:5000/setup-admin` to create the first two admin accounts.
- Once admins exist, `/setup-admin` is disabled.

### Main URLs
- Public display: `http://localhost:5000/display`
- Admin login: `http://localhost:5000/login`
- Admin dashboard: `http://localhost:5000/admin`
- User registration page: `http://localhost:5000/register`
- Reports page: `http://localhost:5000/reports`

## Configuration
Edit values in `config.py`:
- `SECRET_KEY` - Flask session key (set a fixed secure value in production).
- `DEBUG` - Flask debug mode.
- `RFID_ENABLED` / `RFID_SCAN_INTERVAL` - scanner behavior.
- `AUTO_CHECKOUT_TIME` / `AUTO_CHECKOUT_ENABLED` - defaults for auto-checkout behavior.
- `MAX_OCCUPANCY` - default occupancy setting.

> Note: Settings persisted in the `settings` table are read/updated at runtime through API/database functions.

## Database Schema
SQLite database file: `attendance.db`

### `users`
Stores card-to-student mapping and profile data.
- `id` (PK)
- `rfid_uid` (UNIQUE)
- `name`
- `student_id` (UNIQUE)
- `email`
- `graduating_year`
- `assigned_task`
- `is_approved`
- `created_at`

### `checkins`
Stores attendance sessions.
- `id` (PK)
- `user_id` (FK → `users.id`)
- `check_in_time`
- `check_out_time` (nullable while user is currently checked in)
- `auto_checkout` (boolean)

### `admins`
Stores admin login accounts.
- `id` (PK)
- `username` (UNIQUE)
- `password_hash`
- `full_name`
- `created_at`

### `settings`
Simple key-value configuration table.
- `key` (PK)
- `value`

## API Reference
All `/api/*` routes return JSON.

### User management
- `GET /api/users` (auth required) - list users.
- `POST /api/users` (auth required) - create user.
- `PUT /api/users/<user_id>` (auth required) - update user.
- `DELETE /api/users/<user_id>` (auth required) - delete user and related check-ins.

### Attendance/check-in
- `GET /api/checkin/current` - list currently checked-in users.
- `POST /api/checkin/manual` (auth required) - manual `checkin` / `checkout`.
- `POST /api/checkin/scan-card` (auth required) - trigger single card read for registration.

### Reports
- `GET /api/reports/daily?date=YYYY-MM-DD` (auth required)
- `GET /api/reports/weekly` (auth required)
- `GET /api/reports/user/<user_id>` (auth required)

### Settings
- `GET /api/settings` (auth required)
- `POST /api/settings` (auth required)
- `POST /api/auto-checkout` (auth required) - manually auto-checkout all active users.

## Deployment Notes (Raspberry Pi)
- Use `setup.sh` as a guided installer for Pi environments.
- It can configure dependencies, virtualenv, and optional `systemd` services:
  - `attendance-web.service`
  - `attendance-rfid.service`

For production, make sure to:
- set a stable secure `SECRET_KEY`,
- disable debug mode,
- run behind a proper process manager/reverse proxy,
- secure network access to admin routes.

## Troubleshooting
- **`PN532 library not found`**
  - Scanner will run in simulation mode.
  - Install PN532 dependencies and verify I2C wiring.
- **Cannot log in**
  - Ensure admin accounts were created at `/setup-admin`.
- **Card scans do nothing**
  - Confirm card UID exists in `users` and scanner has hardware access.
- **Users remain checked in overnight**
  - Verify `auto_checkout_enabled` and `auto_checkout_time` settings.
