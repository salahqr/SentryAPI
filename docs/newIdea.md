# Product Requirements Document

## HR & Attendance System (with Face-Recognition Check-In)


## 1. Overview

A simple HR management system with two roles only — **HR** and **Employee**. "Employee" covers every level (intern, staff, manager, CEO, etc.) — job title/level is just a profile field, not a separate permission tier. The system handles employee records, attendance (via face recognition), leave requests, strikes, and basic payroll.

## 2. Roles

| Role | Description |
|---|---|
| **HR** | Manages the whole system — employees, attendance, leave, strikes, payroll |
| **Employee** | Any level (intern → CEO). Checks in/out, views own data, requests leave |

## 3. Core Features

- **Auth** — HR creates all accounts; login by role (HR / Employee)
- **Employee Profiles** — name, department, job title/level, salary; HR has full CRUD, employee edits own basic info
- **Face Enrollment** — one-time face capture on first login, stored as an embedding
- **Attendance** — face check-in/check-out; auto-marked Present / Late / Absent against a configured start time; HR can manually correct
- **Attendance Stats** — HR sees late/absent counts per employee, filterable by department; employee sees own history only
- **Strikes** — HR issues a strike (reason + date); employee sees own strikes
- **Employee Directory** — HR searches by name, filters by department (Employee has no access)
- **Leave Requests** — employee requests day off / medical / normal leave; HR approves or rejects; leave balance tracked
- **Payroll** — base salary minus absence deductions; HR runs payroll; employee views/downloads own payslip

## 4. Role & Permission Matrix

| Capability | HR | Employee |
|---|---|---|
| Manage employees | Full | — |
| Own profile | Edit anyone | Edit own only |
| Face enrollment | Manage all | Own only |
| Attendance | View/correct all | View own only |
| Strikes | Issue | View own only |
| Leave | Approve/reject | Request only |
| Payroll | Run + view all | View own payslip |
| Dashboard | Company-wide | Personal only |

---

## 5. Tech Stack

| Layer | Technology |
|---|---|
| Backend API | Laravel (PHP) — Sanctum for auth, Spatie for roles/permissions |
| Frontend | React + Tailwind CSS |
| Database | MySQL |
| Face Recognition | Separate Python (FastAPI/Flask) microservice using a pretrained Hugging Face face-embedding model; Laravel calls it over HTTP |
| PDF (payslips) | DomPDF (Laravel package) |
| Deployment | Docker Compose — Laravel, MySQL, and the Python service as separate containers |

**Why a separate Python service:** face-embedding models aren't run natively in PHP, so the recognition logic lives in its own small service. Laravel sends it an image, gets back a match result, and stores only the embedding + match log in MySQL — not raw face images.
