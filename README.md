# Job Board Platform

A full-stack job board built with Express, SQLite, and a responsive browser interface. Candidates can search, save roles, track applications, manage resumes, and create in-app job alerts. Employers can manage a company profile, publish jobs, filter applicants, review resumes, and update application statuses. Administrators can review platform statistics and search the account directory.

## Requirements and setup on Windows

- Node.js 20 or newer
- npm

Open PowerShell in the project folder (`D:\CODEALPHA_JOB_BOARD_PLATFORM`) and run:

```powershell
npm.cmd install
Copy-Item .env.example .env
notepad .env
```

In Notepad, change `JWT_SECRET` to a private random string of at least 32 characters. Set `ADMIN_EMAIL` and `ADMIN_PASSWORD` to your own values; the admin password must have at least 12 characters. Save the file. Do not include quotes around the values. Then, in the same PowerShell window, run:

```powershell
npm.cmd run seed:admin
npm.cmd run dev
```

The first command creates the admin account, or updates its password if that email already belongs to an admin. It will not change a regular account into an admin. The second command starts the API; leave that window open while using it. The API listens on `http://localhost:3000`. To verify it from a second PowerShell window, run:

```powershell
Invoke-RestMethod http://localhost:3000/api/health
```

It should return `status: ok` and `database: ok`. Open **http://localhost:3000/** in your browser to use the website. Create an account with **Get started**, then choose whether you are looking for work or hiring. Candidates can save jobs, create alerts, upload resumes, and track applications in **My dashboard**. Employers can update their company profile, post jobs, and filter applicants. Admins can review statistics and search the user directory. SQLite tables are created automatically at startup. Keep `DATABASE_PATH` and `UPLOAD_DIR` on persistent storage; uploaded resumes are intentionally excluded from version control.

### Preview with sample jobs

For local UI testing, run `npm.cmd run seed:demo` before starting the server. It adds three sample job listings under **Northstar Demo Company**; rerunning it refreshes and reopens those demo listings without creating duplicates. The demo employer login is intentionally unavailable. To take the sample jobs out of public search later, run `npm.cmd run seed:demo:hide`; this closes only those three demo listings and preserves accounts and any application history. Demo seeding is blocked when `NODE_ENV=production`.

### Local operations

- Create a consistent SQLite backup at any time with `npm.cmd run backup`. Backups are written to `backups\` with a timestamp; provide a different destination path as an argument if needed, for example `npm.cmd run backup -- D:\safe-copies\jobs.sqlite`.
- Production-mode HTTP requests are logged as JSON with a request ID, method, path, status, and duration. Query strings, credentials, and request bodies are not logged.
- The server validates its port and upload-size configuration, exposes a database-aware health check, and closes the server and database cleanly on Ctrl+C or termination.
- Job alerts create in-app notifications when a newly posted role matches the saved alert. Email alerts are not sent.

## Gmail password-reset email setup

Password reset sends a single-use link that expires after one hour. Before users can request links, configure Gmail SMTP on the computer running the API:

1. Enable 2-Step Verification on the Gmail account you will use to send email.
2. Create a Google **App Password** for the app. Use this 16-character app password, not your normal Gmail password. Google may require a work/school administrator or account-security settings to allow app passwords.
3. In `.env`, set the following values (replace both email addresses and the app-password placeholder; do not commit this file):

```dotenv
APP_BASE_URL=http://localhost:3000
SMTP_HOST=smtp.gmail.com
SMTP_PORT=465
SMTP_SECURE=true
SMTP_USER=you@gmail.com
SMTP_PASSWORD=your-16-character-app-password
MAIL_FROM=Job Board Platform <you@gmail.com>
```

4. Save `.env` and restart the server (`Ctrl+C`, then `npm.cmd run dev`).
5. On the sign-in dialog, choose **Forgot your password?**, enter the account email, and check the inbox and spam folder. Open the link, enter and confirm a new password, then sign in with it.

The application intentionally does not send reset email until valid SMTP settings are present. Reset links contain a random one-time token; only its hash is stored in the database. A successful reset invalidates earlier login tokens. For a deployed HTTPS site, change `APP_BASE_URL` to its HTTPS origin.

### Troubleshooting on Windows

- If PowerShell says `npm.ps1 cannot be loaded because running scripts is disabled`, use the `npm.cmd` commands shown above instead of `npm`.
- If Edge displays “Failed to fetch,” make sure `npm.cmd run dev` is still running and open `http://localhost:3000/`. Do not open `public\index.html` directly from File Explorer; the page needs the server to reach the API.
- If `Copy-Item` says `.env` already exists, keep that file and check that its `JWT_SECRET` is at least 32 characters and its admin password is at least 12 characters.
- If startup reports `JWT_SECRET must be configured`, make sure you saved `.env` in the project folder and changed the `JWT_SECRET` line. Do not put quotes around the value.
- If port 3000 is already in use, change `PORT=3000` to another port, such as `PORT=3001`, in `.env`. Use that port in the health-check URL.
- If it still fails, copy the complete red error text from PowerShell, including the command you ran.

## Authentication and roles

Register with `POST /api/auth/register` using role `candidate` or `employer`, or create the first administrator with the seed command. Login returns a bearer token. Send it on protected requests as `Authorization: Bearer <token>`. Public registration cannot create admin accounts.

### Example registration requests

```json
{
  "role": "candidate",
  "email": "candidate@example.com",
  "password": "a-strong-password",
  "fullName": "Taylor Candidate",
  "location": "Austin, TX"
}
```

```json
{
  "role": "employer",
  "email": "jobs@example.com",
  "password": "a-strong-password",
  "companyName": "Example Company",
  "website": "https://example.com"
}
```

## API routes

| Method | Path | Access | Purpose |
| --- | --- | --- | --- |
| `GET` | `/` | Public | Serve the Job Board Platform web app |
| `GET` | `/api/health` | Public | Check application and database availability |
| `POST` | `/api/auth/register` | Public | Create a candidate or employer account |
| `POST` | `/api/auth/login` | Public | Authenticate and receive a token |
| `POST` | `/api/auth/forgot-password` | Public | Send a time-limited password reset link by email |
| `POST` | `/api/auth/reset-password` | Public | Set a new password using a valid reset token |
| `GET` | `/api/me` | Authenticated | View own account and profile |
| `PATCH` | `/api/employer/profile` | Employer | Update company profile |
| `GET` | `/api/jobs` | Public / optional candidate auth | Search open jobs; candidate responses include saved state |
| `GET` | `/api/jobs/:id` | Public / owner / admin | View open job details; owners and admins may also view closed jobs |
| `POST` | `/api/jobs` | Employer | Post a job |
| `GET` | `/api/employer/jobs` | Employer | List own jobs |
| `GET` | `/api/saved-jobs` | Candidate | List saved jobs |
| `POST` | `/api/saved-jobs` | Candidate | Save an open job |
| `DELETE` | `/api/saved-jobs/:jobId` | Candidate | Remove a saved job |
| `GET` | `/api/job-alerts` | Candidate | List saved job alerts |
| `POST` | `/api/job-alerts` | Candidate | Create an in-app matching alert |
| `DELETE` | `/api/job-alerts/:id` | Candidate | Remove a job alert |
| `PATCH` | `/api/jobs/:id` | Owner / admin | Edit a job, or set its status to `open` / `closed` |
| `DELETE` | `/api/jobs/:id` | Owner / admin | Close a job without deleting its application history |
| `POST` | `/api/resumes` | Candidate | Upload a resume as multipart field `resume` |
| `GET` | `/api/resumes` | Candidate | List own resumes |
| `DELETE` | `/api/resumes/:id` | Candidate owner | Delete an unused resume |
| `GET` | `/api/resumes/:id/download` | Owner / relevant employer / admin | Download a resume |
| `POST` | `/api/jobs/:id/applications` | Candidate | Apply to an open job |
| `GET` | `/api/applications` | Candidate | Track own applications |
| `GET` | `/api/employer/applications` | Employer | Review and filter applications to own jobs |
| `GET` | `/api/applications/:id` | Application parties / admin | View an application |
| `PATCH` | `/api/applications/:id/status` | Employer / admin | Update application status |
| `GET` | `/api/notifications` | Authenticated | Read in-app notifications |
| `PATCH` | `/api/notifications/:id/read` | Notification owner | Mark a notification as read |
| `GET` | `/api/admin/stats` | Admin | View platform and application statistics |
| `GET` | `/api/admin/users` | Admin | Paginated user listing |

Job search accepts `q`, `location`, `category`, `employmentType`, `remote`, `salaryMin`, `page`, and `limit` query parameters. For example: `/api/jobs?q=designer&location=remote&employmentType=full-time`.

Application statuses are `submitted`, `reviewing`, `shortlisted`, `rejected`, and `hired`. Employers can change or reset an application status from their dashboard. Employers receive a stored in-app notification on a new application; candidates receive one when an employer changes its status. Duplicate applications to the same job are rejected. Closed jobs can be reopened from the employer dashboard; reopening makes the listing visible to candidates again.

Resumes are limited to PDF, DOC, and DOCX, with a configurable size limit (`MAX_UPLOAD_MB`, default 5 MB). Files are accessible only to their owner, an employer with an application from that candidate, or an admin. Candidates can delete unused resumes from their dashboard; a resume attached to an application cannot be deleted while that application exists, so the employer can still review it.

## Data model

SQLite contains `users`, `candidates`, `employers`, `jobs`, `resumes`, `applications`, and `notifications` tables. Foreign keys are enabled; deleting a job cascades to its applications, while the API's job delete operation closes the job to preserve history.
