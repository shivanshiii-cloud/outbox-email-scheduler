## EMAIL  SCHEDULER 

A full-stack email scheduling application that allows users to compose emails and schedule them for future delivery.

The application uses Express.js, BullMQ, Redis, and a SQL database on the backend, with a React/Next.js dashboard on the frontend. Emails are delivered through Ethereal , a fake SMTP service used for development and testing.



##  Features

### Backend

*  Schedule emails for a future date and time
*  BullMQ delayed jobs for reliable scheduling
*  Persistent job queue using Redis
*  Email and user data stored in a SQL database
*  Per-sender hourly rate limiting
*  Configurable worker concurrency
*  Automatic retries with backoff
*  Email delivery using Nodemailer and Ethereal SMTP
*  Cancel scheduled emails
*  View scheduled and sent emails
*  Scheduled jobs survive API and worker restarts

### Frontend

*  User authentication
*  Dashboard with scheduled and sent email sections
*  Email composition form
*  Schedule emails for a specific date and time
*  Support for multiple recipients
*  Tables for scheduled and sent emails
*  Loading and empty states


### Tech Stack

### Backend

* Node.js
* Express.js
* BullMQ
* Redis
* Nodemailer
* Ethereal Email
* MySQL / PostgreSQL
* JWT / Google OAuth 

### Frontend

* React.js / Next.js
* JavaScript
* HTML & CSS



##  Project Structure

text
Email-Scheduler/
│
├── backend/
│   ├── ...
│   └── ...
│
└── frontend/
    ├── ...
    └── ...


### Backend

The backend contains:

* Express API
* BullMQ queue
* Background worker
* Database layer
* Authentication
* Email scheduling and delivery logic

### Frontend

The frontend contains:

* Login/authentication
* Email composition
* Scheduled email dashboard
* Sent email dashboard
* Email status tables



## 📋 Prerequisites

Make sure you have the following installed:

* Node.js 18+
* Redis 6+
* MySQL 8 / PostgreSQL
* An Ethereal Email account



##  Redis Setup

You can quickly start Redis using Docker:

docker run -d --name redis -p 6379:6379 redis:7


Verify that Redis is running:

docker ps


For production usage, Redis persistence using  AOF or RDB should be enabled so queued jobs can survive a Redis restart.



##  Ethereal Email Setup

Ethereal provides a fake SMTP server for development and testing.

1. Go to  https://ethereal.email
2. Create an Ethereal account.
3. Copy the following SMTP credentials:

   * SMTP host
   * SMTP port
   * Username
   * Password
4. Add them to your backend  ".env " file.

You can also generate an account programmatically using:

javascript:
nodemailer.createTestAccount()


### Viewing Emails

Emails sent through Ethereal **do not reach real inboxes**.

You can view them through:

* The Ethereal Messages page
* The preview URL returned by Nodemailer:

javascript:
nodemailer.getTestMessageUrl(info)




# ⚙️ Environment Variables

## Backend

Create:

```text
backend/.env
```

```env
PORT=4000

DATABASE_URL=mysql://user:password@localhost:3306/email_scheduler

REDIS_HOST=127.0.0.1
REDIS_PORT=6379

SMTP_HOST=smtp.ethereal.email
SMTP_PORT=587
SMTP_USER=your_ethereal_user
SMTP_PASS=your_ethereal_pass

WORKER_CONCURRENCY=5
MAX_EMAILS_PER_HOUR=100
MIN_DELAY_BETWEEN_EMAILS_MS=2000

JWT_SECRET=change_me

GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
```

> Update `DATABASE_URL` and authentication variables according to your actual implementation.

---

## Frontend

Create:

```text
frontend/.env
```

For Vite:

```env
VITE_API_URL=http://localhost:4000
```

For Next.js:

```env
NEXT_PUBLIC_API_URL=http://localhost:4000
```

---

# 🚀 Installation & Setup

## 1. Clone the Repository

```bash
git clone <your-repository-url>
cd Email-Scheduler
```

---

## 2. Start the Backend

```bash
cd backend
npm install
```

Create your environment file:

```bash
cp .env.example .env
```

Update the values inside `.env`.

Run your database migrations:

```bash
npm run db:migrate
```

> Replace this command with the migration command used by your project, such as Prisma, Knex, or Sequelize.

Start the Express API:

```bash
npm run dev
```

The API will run on:

```text
http://localhost:4000
```

---

## 3. Start the Worker

Open a **second terminal**:

```bash
cd backend
npm run worker
```

The API server and BullMQ worker run as separate processes.

Both processes require:

* Redis
* Database
* Correct environment variables

---

## 4. Start the Frontend

Open another terminal:

```bash
cd frontend
npm install
npm run dev
```

For Vite, the application will typically run at:

```text
http://localhost:5173
```

For Next.js:

```text
http://localhost:3000
```

---

#  Architecture

```text
                    ┌─────────────────┐
                    │    Frontend     │
                    │  React / Next.js│
                    └────────┬────────┘
                             │
                            HTTP
                             │
                             ▼
                    ┌─────────────────┐
                    │   Express API   │
                    └───────┬─────────┘
                            │
                 ┌──────────┴──────────┐
                 │                     │
                 ▼                     ▼
          ┌──────────────┐      ┌──────────────┐
          │  SQL Database│      │    BullMQ    │
          │              │      │    Queue     │
          └──────────────┘      └──────┬───────┘
                                       │
                                     Redis
                                       │
                                       ▼
                              ┌─────────────────┐
                              │     Worker      │
                              │    BullMQ       │
                              └────────┬────────┘
                                       │
                                     SMTP
                                       │
                                       ▼
                              ┌─────────────────┐
                              │ Ethereal Email  │
                              └─────────────────┘
```

---

# How Email Scheduling Works

When a user schedules an email:

1. The frontend sends the email details and `sendAt` time to the API.
2. The API stores the email in the database with a `scheduled` status.
3. A delayed BullMQ job is created.
4. The delay is calculated as:

```text
sendAt - current time
```

5. BullMQ stores the delayed job in Redis.
6. When the scheduled time arrives, BullMQ moves the job to the waiting queue.
7. The worker picks up the job.
8. Nodemailer sends the email through Ethereal SMTP.
9. The database status is updated to either:

   * `sent`
   * `failed`

No `setTimeout()` or traditional cron job is required for scheduling.

---

#  Persistence & Restart Handling

The system is designed so that scheduling does not depend on server memory.

* **Redis** stores the BullMQ jobs.
* **SQL database** stores the source of truth for email state.
* Scheduled jobs remain available when the API or worker restarts.
* Jobs that are already past their scheduled time can be processed when the worker starts.
* Deterministic job IDs help prevent the same email from being enqueued multiple times.

Example:

```text
email-123
```

where `123` represents the corresponding database ID.

If implemented, a startup reconciliation process can also check the database for scheduled emails that are missing from Redis and re-enqueue them.

---

# 🚦 Rate Limiting & Concurrency

## Concurrency

The worker supports configurable concurrency:

```env
WORKER_CONCURRENCY=5
```

This allows multiple email jobs to be processed in parallel.

Multiple worker processes can also be run, with BullMQ/Redis coordinating job processing.

---

## Rate Limiting

The application supports a per-sender hourly limit:

```env
MAX_EMAILS_PER_HOUR=100
```

A Redis counter can be used to track the number of emails sent within an hourly window.

For example:

```text
sender + current hour → Redis counter
```

When the limit is reached, emails are **not dropped**.

Instead, they can be delayed and retried during the next available sending window.

---

## Minimum Delay Between Emails

To prevent sudden bursts of emails:


MIN_DELAY_BETWEEN_EMAILS_MS=2000


This introduces a minimum gap between consecutive email sends.

---

# Retries & Failure Handling

Email jobs can be configured with retries and exponential/backoff delays.

If an email fails:


Job
 │
 ├── Attempt 1 → Failed
 │
 ├── Retry
 │
 ├── Attempt 2 → Failed
 │
 └── Retry
       │
       ▼
     Sent / Failed


The final status is stored in the database.

---

# Bulk Email Scheduling

For bulk campaigns, each recipient can be represented as a separate BullMQ job.


Campaign
   │
   ├── Recipient 1 → Job 1
   ├── Recipient 2 → Job 2
   ├── Recipient 3 → Job 3
   └── Recipient 4 → Job 4


Jobs can be staggered to spread email delivery over time and respect rate limits.



#  API

The backend provides endpoints for operations such as:

* Schedule an email
* List scheduled emails
* List sent emails
* Cancel a scheduled email
* Authentication

### API Routes(Examples)


POST   /api/emails/schedule
GET    /api/emails/scheduled
GET    /api/emails/sent
DELETE /api/emails/:id


#  Frontend Dashboard

The dashboard provides a central interface for managing scheduled emails.

### Login

Users can authenticate using:

> Add your actual authentication method here.

For example:

* Google OAuth
* JWT-based authentication

### Dashboard

The dashboard contains sections for:


Dashboard
│
├── Scheduled Emails
│
├── Sent Emails
│
└── Compose Email


### Compose Email

Users can provide:

* Recipients
* Subject
* Email body
* Scheduled send time
* Optional delay
* Hourly sending limit

### Email Tables

The dashboard displays information such as:

* Recipient
* Subject
* Status
* Scheduled time
* Sent time

Loading and empty states are also handled.





Frontend
   │
   ▼
Login
   │
   ▼
Authentication Provider
   │
   ▼
JWT
   │
   ▼
Express API


#  Development

The application requires three main processes during development:

### Terminal 1 — Redis


Docker start redis


### Terminal 2 — Backend API


cd backend
npm run dev


### Terminal 3 — BullMQ Worker


cd backend
npm run worker


### Terminal 4 — Frontend


--cd frontend
--npm run dev




### Assumptions & Trade-offs

### Ethereal SMTP

Ethereal is used instead of a production email provider.

This means:

* Emails are not delivered to real inboxes.
* Emails can be inspected through the Ethereal dashboard.
* The project can be tested without sending real emails.

### Redis Persistence

Since BullMQ relies on Redis for queued jobs, Redis persistence should be enabled in environments where scheduled jobs must survive a Redis restart.

### Future Improvements

Potential improvements include:

* Dead-letter queue
* Per-user rate limits
* Better retry monitoring
* Automated tests
* Email templates
* File/CSV recipient uploads
* Email attachments
* Production SMTP provider
* Improved campaign management
* Detailed worker/job monitoring



#   Key Design Decisions

| Requirement           | Implementation                  |
| --------------------- | ------------------------------- |
| Email scheduling      | BullMQ delayed jobs             |
| Queue storage         | Redis                           |
| Email state           | SQL database                    |
| Email delivery        | Nodemailer + Ethereal           |
| Background processing | BullMQ Worker                   |
| Concurrency           | Configurable worker concurrency |
| Rate limiting         | Redis counter / BullMQ limiter  |
| Retry handling        | BullMQ retries + backoff        |
| Frontend              | React / Next.js                 |
| Backend               | Express.js                      |



