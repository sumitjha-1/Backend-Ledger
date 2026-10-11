# Backend Ledger

A Node.js and Express REST API for a ledger-based account system. It supports user registration and login, account creation, balance calculation, account-to-account transfers, system-user initial funding, JWT authentication, and email notifications.

## Features

- User registration and login with password hashing using `bcryptjs`.
- JWT authentication through an HTTP cookie or an `Authorization: Bearer <token>` header.
- Logout using a token blacklist.
- Account creation and account listing for the authenticated user.
- Balance calculation from immutable credit and debit ledger entries.
- Account-to-account transactions with an idempotency key.
- MongoDB transactions for grouping transfer writes.
- A protected endpoint for a system user to provide initial funds.
- Registration and transaction email notifications using Nodemailer and Gmail OAuth2.

## Tech Stack

- Node.js
- Express
- MongoDB and Mongoose
- JSON Web Tokens (`jsonwebtoken`)
- `bcryptjs`
- `cookie-parser`
- Nodemailer
- `dotenv`

## Project Structure

```text
backend-ledger-main/
├── server.js
└── src/
    ├── app.js
    ├── config/
    │   └── db.js
    ├── controllers/
    │   ├── account.controller.js
    │   ├── auth.controller.js
    │   └── transaction.controller.js
    ├── middleware/
    │   └── auth.middleware.js
    ├── models/
    │   ├── account.model.js
    │   ├── blackList.model.js
    │   ├── ledger.model.js
    │   ├── transaction.model.js
    │   └── user.model.js
    ├── routes/
    │   ├── account.routes.js
    │   ├── auth.routes.js
    │   └── transaction.routes.js
    └── services/
        └── email.service.js
```

## Prerequisites

- Node.js and npm
- MongoDB. The transaction endpoints use MongoDB sessions/transactions, so use a MongoDB replica set or a compatible MongoDB deployment that supports transactions.
- Gmail OAuth2 credentials if email notifications are required.

## Installation

1. Clone or download this repository.
2. Open a terminal in the project directory.
3. Install the required packages:

   ```bash
   npm install express mongoose dotenv cookie-parser jsonwebtoken bcryptjs nodemailer
   ```

4. Create a `.env` file in the project root.
5. Configure the environment variables below.
6. Start the server.

If the repository includes a `package.json`, prefer `npm install` to install its declared dependencies.

## Environment Variables

Create `.env` in the same directory as `server.js`:

```env
MONGO_URI=mongodb://127.0.0.1:27017/backend-ledger
JWT_SECRET=replace_with_a_long_random_secret

EMAIL_USER=your-email@gmail.com
CLIENT_ID=your-google-oauth-client-id
CLIENT_SECRET=your-google-oauth-client-secret
REFRESH_TOKEN=your-google-oauth-refresh-token
```

Use a strong, unique `JWT_SECRET`. Keep `.env` out of version control and never commit real credentials. Email credentials are needed for the configured Gmail OAuth2 transporter; the API's email service logs connection/send errors if email delivery fails.

## Run the Server

The server listens on port `3000`.

```bash
node server.js
```

The root health route is:

```http
GET /
```

Expected response:

```text
Ledger Service is up and running
```

## API Reference

Base URL for local development: `http://localhost:3000`

### Authentication

| Method | Endpoint | Authentication | Purpose |
|---|---|---|---|
| `POST` | `/api/auth/register` | Not required | Register a user |
| `POST` | `/api/auth/login` | Not required | Log in |
| `POST` | `/api/auth/logout` | Token used if present | Log out and blacklist the token |

#### Register

`POST /api/auth/register`

```json
{
  "name": "Aman",
  "email": "aman@example.com",
  "password": "your-password"
}
```

A successful request returns the user's ID, name, email, and JWT, and sets a `token` cookie.

#### Login

`POST /api/auth/login`

```json
{
  "email": "aman@example.com",
  "password": "your-password"
}
```

A successful request returns the user details and JWT, and sets a `token` cookie.

#### Logout

`POST /api/auth/logout`

The server reads the token from the `token` cookie or the `Authorization` header, adds it to the blacklist, and clears the cookie.

### Accounts

All account endpoints require authentication.

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/accounts` | Create an account for the logged-in user |
| `GET` | `/api/accounts` | List the logged-in user's accounts |
| `GET` | `/api/accounts/balance/:accountId` | Get an account's balance |

Example request to create an account:

```http
POST /api/accounts
Authorization: Bearer <token>
```

Example balance response:

```json
{
  "accountId": "ACCOUNT_ID",
  "balance": 1500
}
```

Balances are calculated from ledger entries: total credits minus total debits. A new account has a balance of `0` until it receives a credit.

### Transactions

All transaction endpoints require authentication. Send JSON request bodies with `Content-Type: application/json`.

| Method | Endpoint | Authentication | Purpose |
|---|---|---|---|
| `POST` | `/api/transactions` | Regular authenticated user | Transfer funds between accounts |
| `POST` | `/api/transactions/system/initial-funds` | Authenticated system user | Fund a recipient account from the system account |

#### Transfer funds

`POST /api/transactions`

```json
{
  "fromAccount": "SENDER_ACCOUNT_ID",
  "toAccount": "RECIPIENT_ACCOUNT_ID",
  "amount": 250,
  "idempotencyKey": "unique-request-key-001"
}
```

The transfer flow checks the accounts and their status, checks the idempotency key, calculates the sender's ledger-derived balance, creates a pending transaction, writes debit and credit ledger entries, marks the transaction completed, and commits the MongoDB session.

The `idempotencyKey` must be unique. Reusing a key for an already completed transaction returns the existing transaction rather than intentionally creating another one.

#### Add initial funds

`POST /api/transactions/system/initial-funds`

```json
{
  "toAccount": "RECIPIENT_ACCOUNT_ID",
  "amount": 1000,
  "idempotencyKey": "initial-funds-001"
}
```

This route uses `authSystemUserMiddleware`. The authenticated user must have `systemUser: true` in the database. The current project structure does not provide a public API for creating or promoting a system user; configure one through a trusted administrative process.

## Authentication

Send the JWT in either of these ways:

```http
Authorization: Bearer <token>
```

or use the `token` cookie set by registration and login. Tokens expire after three days. Logout blacklists a token, and blacklist documents are configured to expire after three days.

## Data Model Overview

- **User:** name, email, hashed password, and system-user flag.
- **Account:** owner, status (`ACTIVE`, `FROZEN`, or `CLOSED`), and currency (defaults to `INR`).
- **Transaction:** source account, destination account, amount, status, and unique idempotency key.
- **Ledger:** immutable credit/debit entries associated with an account and transaction.
- **Token blacklist:** invalidated JWTs, with a three-day TTL index.

## Important Implementation Notes

- This is a backend project skeleton, not a claim of production-grade banking software. Review and test it before handling real funds.
- MongoDB transactions require a deployment that supports sessions and transactions.
- The transfer controller currently contains a deliberate 15-second delay between writing the debit and credit ledger entries. Remove it if it is only for testing.
- Review transaction error handling: the current catch path does not explicitly abort and end the MongoDB session.
- Before production use, add robust validation for account IDs and amounts, ensure a positive finite amount is required, verify that the sender account belongs to the authenticated user, and make transaction processing safe under concurrent requests.
- Configure secure cookie options appropriate to the deployment (including `httpOnly`, `secure`, and `sameSite`), add rate limiting and centralized error handling, and avoid logging passwords or other sensitive data.
- Email delivery errors are logged by the email service; email failures should be monitored separately from transaction state.

## License

No license is specified in the provided project files. Add a `LICENSE` file if you intend to distribute the project.
