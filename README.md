# P2P Enterprise 💰

> A cloud-native Peer-to-Peer (P2P) lending platform with automated capital allocation, risk-based portfolio diversification, and ACID-compliant transactions.

[![Python](https://img.shields.io/badge/Python-3.11-blue.svg)](https://www.python.org/)
[![Flask](https://img.shields.io/badge/Flask-3.0.0-black.svg)](https://flask.palletsprojects.com/)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-orange.svg)](https://www.mysql.com/)
[![TiDB](https://img.shields.io/badge/TiDB-Cloud-red.svg)](https://tidbcloud.com/)
[![Vercel](https://img.shields.io/badge/Deploy-Vercel-black.svg)](https://vercel.com/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](#license)

---

## 📖 Table of Contents

- [Overview](#-overview)
- [Features](#-features)
- [Tech Stack](#-tech-stack)
- [Architecture](#-architecture)
- [Database Schema](#-database-schema)
- [Getting Started](#-getting-started)
  - [Prerequisites](#prerequisites)
  - [Local Setup](#local-setup)
  - [Environment Configuration](#environment-configuration)
  - [Running the App](#running-the-app)
- [Deployment (Vercel + TiDB Cloud)](#-deployment-vercel--tidb-cloud)
- [API Routes](#-api-routes)
- [Project Structure](#-project-structure)
- [How It Works](#-how-it-works)
  - [Automated Allocation Engine](#automated-allocation-engine)
  - [EMI Routing Simulator](#emi-routing-simulator)
  - [Invoice Generation](#invoice-generation)
- [Testing](#-testing)
- [Security Notes](#-security-notes)
- [Roadmap](#-roadmap)
- [Team](#-team)
- [License](#-license)

---

## 🔍 Overview

**P2P Enterprise** is a full-stack web application that connects **lenders** (investors) directly with **borrowers** (credit seekers), bypassing traditional financial intermediaries. It includes:

- Role-based dashboards for **Admin**, **Lender**, and **Borrower**
- An **automated capital allocation engine** matching lender mandates (ROI range, risk appetite, auto-invest) with open loan requests
- **ACID-compliant transactions** using row-level locking (`SELECT ... FOR UPDATE`) to prevent over-allocation
- **EMI routing simulation** that debits borrowers and credits lenders
- **Capital analytics** and **risk segmentation** for administrators
- **Printable PDF invoices** for every allocation mapping
- **Serverless deployment** on Vercel with **TiDB Cloud** as the database

---

## ✨ Features

### For Borrowers
- ✅ Multi-step registration with KYC (PAN, Aadhaar, address)
- ✅ Create loan requests with amount, tenure, purpose, and expected interest
- ✅ Track funding progress and view list of backers

### For Lenders
- ✅ Configure investment mandate (capital, ROI range, risk appetite, auto-invest toggle)
- ✅ Automatic allocation to matching loans (no manual intervention)
- ✅ View personalized portfolio of funded loans
- ✅ Download allocation invoices

### For Admins
- ✅ Full analytics dashboard (real borrowers, real lenders, mappings)
- ✅ Capital utilization metrics (idle vs. allocated)
- ✅ Risk-appetite segmentation (Conservative / Moderate / Aggressive)
- ✅ Trigger the automated allocation engine
- ✅ Simulate one month of EMI routing across all active mappings
- ✅ Generate audit-ready invoices per transaction

### System-Level
- 🔒 Row-level locking for concurrency safety
- 📊 Real-time diversification ledger
- ☁️ Serverless, zero-ops deployment
- 📱 Responsive UI (desktop + mobile)

---

## 🧰 Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | HTML5, CSS3, Jinja2 Templates |
| **Backend** | Python 3.11, Flask 3.0.0 |
| **Database** | TiDB Cloud (MySQL 8.0 compatible) |
| **DB Driver** | `mysql-connector-python==8.2.0` |
| **Deployment** | Vercel (`@vercel/python` runtime) |
| **Version Control** | Git + GitHub |

---

## 🏗 Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    PRESENTATION TIER                     │
│   HTML5 + CSS3 + Jinja2 Templates (Role-Based Views)     │
└────────────────────────┬─────────────────────────────────┘
                         │ HTTP / HTTPS
┌────────────────────────▼─────────────────────────────────┐
│                    APPLICATION TIER                      │
│           Flask 3.0.0 (Python 3.11 runtime)              │
│  ┌──────────┬──────────┬──────────┬──────────┬─────────┐ │
│  │ /login   │/register │/run_     │/simulate │/invoice │ │
│  │          │_full     │engine    │_emi      │/<id>    │ │
│  └──────────┴──────────┴──────────┴──────────┴─────────┘ │
└────────────────────────┬─────────────────────────────────┘
                         │ mysql-connector-python (TLS)
┌────────────────────────▼─────────────────────────────────┐
│                       DATA TIER                          │
│           TiDB Cloud (MySQL 8.0 compatible)              │
│  Users • KYC_Details • Borrower_Profiles                 │
│  Lender_Profiles • Loan_Requests • Investment_Mapping    │
└──────────────────────────────────────────────────────────┘
```

**Deployment Flow:**

```
GitHub Push  ➜  Vercel Build (@vercel/python)  ➜  Serverless Function  ➜  TiDB Cloud
```

---

## 🗄 Database Schema

```sql
-- Core identity
CREATE TABLE Users (
    UserID    INT AUTO_INCREMENT PRIMARY KEY,
    FullName  VARCHAR(100),
    Email     VARCHAR(100) UNIQUE,
    Mobile    VARCHAR(15),
    Password  VARCHAR(255),           -- ⚠️ hash before production
    UserType  ENUM('borrower','lender')
);

-- KYC records (1:1 with Users)
CREATE TABLE KYC_Details (
    KYCID          INT AUTO_INCREMENT PRIMARY KEY,
    UserID         INT UNIQUE,
    DOB            DATE,
    Gender         VARCHAR(10),
    PAN_Number     VARCHAR(10),
    Aadhaar_Number VARCHAR(12),
    Street         VARCHAR(150),
    City           VARCHAR(50),
    State          VARCHAR(50),
    Pincode        VARCHAR(6),
    FOREIGN KEY (UserID) REFERENCES Users(UserID)
);

-- Borrower profile (1:1 with Users)
CREATE TABLE Borrower_Profiles (
    BorrowerID    INT AUTO_INCREMENT PRIMARY KEY,
    UserID        INT UNIQUE,
    EmpType       VARCHAR(50),
    MonthlyIncome DECIMAL(12,2),
    CompanyName   VARCHAR(100),
    WorkExp       INT,
    ExistingEMI   DECIMAL(12,2),
    CibilScore    INT,
    BankBalance   DECIMAL(12,2) DEFAULT 0,
    FOREIGN KEY (UserID) REFERENCES Users(UserID)
);

-- Lender profile (1:1 with Users)
CREATE TABLE Lender_Profiles (
    LenderID      INT AUTO_INCREMENT PRIMARY KEY,
    UserID        INT UNIQUE,
    InvestAmount  DECIMAL(12,2),
    RiskAppetite  ENUM('conservative','moderate','aggressive'),
    MinROI        DECIMAL(5,2),
    MaxROI        DECIMAL(5,2),
    AutoInvest    ENUM('yes','no'),
    FOREIGN KEY (UserID) REFERENCES Users(UserID)
);

-- Loan requests (N:1 with Borrowers)
CREATE TABLE Loan_Requests (
    LoanID            INT AUTO_INCREMENT PRIMARY KEY,
    UserID            INT,
    Amount_Needed     DECIMAL(12,2),
    Amount_Funded     DECIMAL(12,2) DEFAULT 0,
    Purpose           VARCHAR(150),
    Tenure            INT,
    Expected_Interest DECIMAL(5,2),
    Status            ENUM('OPEN','FILLED') DEFAULT 'OPEN',
    FOREIGN KEY (UserID) REFERENCES Users(UserID)
);

-- Investment mappings (M:N between Lenders and Loans)
CREATE TABLE Investment_Mapping (
    MappingID        INT AUTO_INCREMENT PRIMARY KEY,
    InvestorID       INT,
    LoanID           INT,
    Amount_Allocated DECIMAL(12,2),
    Allocation_Date  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (InvestorID) REFERENCES Users(UserID),
    FOREIGN KEY (LoanID)     REFERENCES Loan_Requests(LoanID)
);
```

---

## 🚀 Getting Started

### Prerequisites

- **Python 3.11+**
- **MySQL 8.0+** (local) **or** a **TiDB Cloud** account (free tier)
- **Git**
- **Vercel CLI** (optional, for deployment)

### Local Setup

```bash
# 1. Clone the repository
git clone https://github.com/<your-username>/p2p-enterprise.git
cd p2p-enterprise

# 2. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate          # Linux / macOS
venv\Scripts\activate             # Windows

# 3. Install dependencies
pip install -r requirements.txt
```

### Environment Configuration

Create a `.env` file (or edit `app.py` directly for local testing):

```env
DB_HOST=gateway01.ap-southeast-1.prod.aws.tidbcloud.com
DB_USER=your_tidb_user.root
DB_PASSWORD=your_tidb_password
DB_NAME=p2p_enterprise
```

Then update `get_db_connection()` in `app.py`:

```python
def get_db_connection():
    return mysql.connector.connect(
        host=os.getenv("DB_HOST"),
        user=os.getenv("DB_USER"),
        password=os.getenv("DB_PASSWORD"),
        database=os.getenv("DB_NAME")
    )
```

### Running the App

```bash
python app.py
```

Visit **http://127.0.0.1:5000** in your browser.

**Admin login (demo):**
- Email: `admin@gmail.com`
- Password: `admin`

---

## ☁️ Deployment (Vercel + TiDB Cloud)

### Step 1 — Set up TiDB Cloud

1. Sign up at [tidbcloud.com](https://tidbcloud.com)
2. Create a **Serverless** cluster (free tier available)
3. Copy the connection details (host, user, password)
4. Run the SQL schema above in the TiDB Cloud SQL Editor

### Step 2 — Push to GitHub

```bash
git init
git add .
git commit -m "Initial commit: P2P Enterprise"
git branch -M main
git remote add origin https://github.com/<your-username>/p2p-enterprise.git
git push -u origin main
```

### Step 3 — Deploy on Vercel

```bash
# Install Vercel CLI
npm install -g vercel

# Login and deploy
vercel login
vercel --prod
```

Or use the Vercel dashboard:

1. Go to [vercel.com/new](https://vercel.com/new)
2. Import your GitHub repository
3. Set **Framework Preset** to `Other`
4. Add environment variables (`DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`)
5. Click **Deploy**

**Required `vercel.json`:**

```json
{
  "version": 2,
  "builds": [
    {
      "src": "app.py",
      "use": "@vercel/python",
      "config": { "includeFiles": ["templates/**", "static/**"] }
    }
  ],
  "routes": [
    { "src": "/(.*)", "dest": "app.py" }
  ]
}
```

---

## 🛣 API Routes

| Method | Route | Description | Access |
|--------|-------|-------------|--------|
| `GET`  | `/` | Landing page | Public |
| `POST` | `/login` | Authenticate (Admin / Lender / Borrower) | Public |
| `POST` | `/register_full` | Full registration + KYC + profile + loan request | Public |
| `POST` | `/run_engine` | Trigger automated capital allocation engine | Admin |
| `POST` | `/simulate_emi` | Simulate 1 month of EMI routing | Admin |
| `GET`  | `/invoice/<mapping_id>` | Generate printable allocation invoice | Admin / Lender |

---

## 📁 Project Structure

```
p2p-enterprise/
│
├── app.py                     # Main Flask application
├── requirements.txt           # Python dependencies
├── vercel.json                # Vercel deployment config
├── README.md                  # This file
├── LICENSE                    # MIT License
│
├── templates/                 # Jinja2 HTML templates
│   ├── index.html             # Landing + role-based dashboards
│   ├── login.html
│   ├── register.html
│   └── ...
│
└── static/                    # CSS, JS, images
    ├── css/
    │   └── style.css
    └── js/
        └── main.js
```

---

## ⚙️ How It Works

### Automated Allocation Engine

Triggered by `POST /run_engine`. For each lender with `AutoInvest='yes'` and `InvestAmount > 0`:

1. Finds the first `OPEN` loan where `Expected_Interest >= MinROI` using `SELECT ... FOR UPDATE`
2. Computes `allocated = min(gap, InvestAmount)`
3. Inserts an `Investment_Mapping` row
4. Updates `Loan_Requests.Amount_Funded` and `Lender_Profiles.InvestAmount`
5. Marks loan as `FILLED` when fully funded
6. Commits (or rolls back if no match)

**Concurrency safety:** `FOR UPDATE` locks the loan row so no two lenders can fund the same gap simultaneously.

### EMI Routing Simulator

Triggered by `POST /simulate_emi`. For every mapping where the loan status is `FILLED`:

```
EMI = (Principal + Principal × Interest) / Tenure
```

Funds are **debited** from `Borrower_Profiles.BankBalance` and **credited** to `Lender_Profiles.InvestAmount`.

> ⚠️ The current formula uses **flat interest**. A reducing-balance formula `EMI = P·i·(1+i)^T / ((1+i)^T - 1)` is a planned enhancement.

### Invoice Generation

Triggered by `GET /invoice/<mapping_id>`. Performs a 4-table join to fetch investor + borrower + allocation details and renders a print-friendly HTML invoice with a `window.print()` button for PDF export.

---

## 🧪 Testing

### Manual Testing

| # | Test Case | Expected Result |
|---|-----------|----------------|
| T1 | Borrower registration with KYC | All 4 rows created (Users, KYC, Borrower_Profiles, Loan_Requests) |
| T2 | Lender registration with ROI range | 3 rows created |
| T3 | Admin login | Dashboard with analytics |
| T4 | Run allocation engine | Investment_Mapping rows created |
| T5 | Two lenders, one loan (concurrent) | No over-allocation |
| T6 | Simulate EMI on 10 mappings | Balances updated atomically |
| T7 | Generate invoice | Printable HTML rendered |
| T8 | Invalid login | "Authentication Failed" |
| T9 | Engine with no open loans | Graceful "no match" log |
| T10 | Registration with missing field | Transaction rolled back |

### Concurrency Test

Use **Apache JMeter** or a shell script with parallel `curl` requests:

```bash
for i in {1..50}; do
  curl -X POST https://your-app.vercel.app/run_engine &
done
wait
```

Verify no loan has `Amount_Funded > Amount_Needed`.

---

## 🔐 Security Notes

> ⚠️ **This project is a demonstration / academic prototype. Do NOT deploy to production without the following fixes:**

| Issue | Current State | Recommended Fix |
|-------|--------------|-----------------|
| Password storage | Plaintext | Use **bcrypt** or **Argon2id** |
| Admin credentials | Hardcoded | Move to environment variables |
| Session management | None | Add **JWT** or signed cookies |
| CSRF protection | None | Add Flask-WTF CSRF tokens |
| Rate limiting | None | Use Flask-Limiter on `/login` |
| Input validation | Minimal | Server-side regex for PAN, Aadhaar, mobile |
| Secrets | In source code | `.env` + Vercel environment variables |

**Already implemented:**
- ✅ SQL injection protection via parameterized queries
- ✅ TLS to TiDB Cloud
- ✅ HTTPS via Vercel

---

## 🗺 Roadmap

- [x] Role-based authentication (Admin / Lender / Borrower)
- [x] KYC-driven registration
- [x] Automated allocation engine with ACID guarantees
- [x] EMI routing simulator
- [x] Capital analytics dashboard
- [x] Invoice generation
- [x] Vercel + TiDB Cloud deployment
- [ ] Password hashing (bcrypt / Argon2)
- [ ] JWT session management
- [ ] ML credit scoring (XGBoost on CIBIL, income, EMI)
- [ ] Secondary market for loan trading
- [ ] Blockchain-based audit trail
- [ ] Email / SMS notifications
- [ ] Payment gateway integration (Razorpay / Stripe)
- [ ] React Native mobile app
- [ ] RBI NBFC-P2P compliance alignment

---

## 👥 Team

| Name | Roll Number | Role |
|------|-------------|------|
| **Om Yewale** | 24101A0069 | Backend & Allocation Engine |
| **Kedar Sawant** | 24101A0070 | Database & Deployment |
| **Advay Tongaonkar** | 24101A0073 | Frontend & Analytics |

**Department of Information Technology**
**Vidyalankar Institute of Technology**, Mumbai, India

---

## 📄 License

This project is licensed under the **MIT License**. See the [LICENSE](LICENSE) file for details.

---

## 🙏 Acknowledgements

- [Flask](https://flask.palletsprojects.com/) — Web framework
- [TiDB Cloud](https://tidbcloud.com/) — Distributed MySQL-compatible database
- [Vercel](https://vercel.com/) — Serverless deployment
- [mysql-connector-python](https://dev.mysql.com/doc/connector-python/en/) — DB driver
- Inspiration: LendingClub, Prosper, Faircent

---

<p align="center">
  <b>⭐ Star this repository if you found it useful! ⭐</b><br>
  Made with ❤️ by the P2P Enterprise Team
</p>
