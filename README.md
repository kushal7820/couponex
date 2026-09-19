[README.md](https://github.com/user-attachments/files/32417769/README.md)
# CouponX — AI-Powered Secure Coupon Marketplace

> A full-stack peer-to-peer coupon marketplace with AES-256 encryption, JusPay escrow payments, AI fraud detection, OTP-based coupon transfer, and automatic refund on validation failure.

---

## Table of Contents

- [Architecture](#architecture)
- [Security Model](#security-model)
- [Escrow Flow](#escrow-flow)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Environment Variables](#environment-variables)
- [API Reference](#api-reference)
- [JusPay Integration](#juspay-integration)
- [AI Features](#ai-features)
- [Deployment](#deployment)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT (React)                           │
│  Pages: Home · Marketplace · Sell · Payment · Dashboard        │
│  Components: CouponCard · ChatWidget · Navbar                  │
│  Services: api.js (axios + auto token refresh)                  │
└─────────────────┬───────────────────────────────────────────────┘
                  │ HTTPS / REST
┌─────────────────▼───────────────────────────────────────────────┐
│                    EXPRESS API (Node.js)                        │
│                                                                 │
│  Middleware Stack:                                              │
│  helmet → cors → rate-limit → mongo-sanitize → hpp → xss       │
│                                                                 │
│  Routes:                                                        │
│  /api/auth      → register · login · refresh · logout · me     │
│  /api/coupons   → list · get · create · delete · my-listings   │
│  /api/payments  → initiate · webhook · verify-otp · status     │
│  /api/chat      → AI support chatbot                           │
└────┬──────────┬──────────┬──────────────────────────────────────┘
     │          │          │
┌────▼───┐  ┌──▼───┐  ┌───▼──────────────┐
│MongoDB │  │JusPay│  │Anthropic Claude  │
│(Mongoose) │ API  │  │(AI scoring+chat) │
└────────┘  └──────┘  └──────────────────┘
```

---

## Security Model

| Layer | Mechanism |
|-------|-----------|
| Coupon codes | AES-256-CBC encrypted at rest — `encryptedCode` field never returned by API |
| Masked display | Only `maskedCode` (e.g. `SAV*****`) is shown before purchase |
| Code reveal | Decrypted **only** inside `escrow.verifyOTP()` after successful OTP match |
| OTP storage | HMAC-SHA256 hashed — plain OTP never persists in DB |
| Passwords | bcrypt (cost factor 12) |
| JWT | Access token (7d) + Refresh token (30d), stored in `localStorage` |
| Account lockout | 5 failed logins → 2-hour lock |
| Rate limiting | Auth: 10/15min · Payments: 20/hr · Chat: 30/min · Global: 300/15min |
| Input sanitization | XSS filter · MongoDB operator injection prevention · HPP protection |
| Headers | Helmet CSP, HSTS, X-Frame-Options, etc. |
| Webhook | HMAC-SHA256 signature verification (JusPay) |

---

## Escrow Flow

```
Buyer clicks "Buy Now"
        │
        ▼
[1] escrow.initiate()
    • Coupon status → "reserved" (locked from other buyers)
    • Transaction created in DB (status: payment_pending)
    • JusPay session created → paymentUrl returned to frontend
        │
        ▼
[2] Buyer pays on JusPay hosted page
        │
        ▼
[3] JusPay → POST /api/payments/webhook (CHARGED event)
    • Signature verified via HMAC-SHA256
    • escrow.onPaymentSuccess() called
    • Money held in escrow, transaction → "in_escrow"
        │
        ▼
[4] escrow.validate() [async]
    • Coupon checked: still exists, not expired, status = "reserved"
    • ❌ Validation FAILS → escrow.abortAndRefund()
      - Coupon → "active" again
      - JusPay refund API called automatically
      - Transaction → "refunded"
    • ✅ Validation PASSES →
      - 6-digit OTP generated (crypto.randomInt)
      - OTP hashed with HMAC-SHA256, stored in coupon.otp.hash
      - OTP sent to buyer (SMS/email in production)
        │
        ▼
[5] Buyer enters OTP on frontend
    POST /api/payments/verify-otp
    • Max 3 attempts — on 3rd failure → auto-refund
    • OTP expiry: 10 minutes — on expiry → auto-refund
    • HMAC compared with timing-safe comparison
    • ❌ Wrong OTP → attempt counter incremented, error returned
    • ✅ OTP correct →
      - coupon.encryptedCode decrypted (AES-256-CBC)
      - Real code returned to buyer (ONLY time it's ever decrypted)
      - Coupon status → "sold"
      - Transaction → "completed", escrow → "released"
      - Seller wallet credited
```

---

## Tech Stack

**Backend**
- Node.js + Express
- MongoDB + Mongoose
- JWT (access + refresh tokens)
- bcryptjs (password hashing)
- AES-256-CBC (coupon encryption)
- HMAC-SHA256 (OTP hashing, webhook verification)
- JusPay (payment gateway)
- Anthropic Claude (AI scoring + chatbot)
- Winston (logging)
- node-cron (coupon expiry jobs)
- helmet, express-rate-limit, hpp, express-mongo-sanitize, xss

**Frontend**
- React 18 + React Router v6
- Axios (with token refresh interceptor)
- react-hot-toast

---

## Project Structure

```
couponx/
├── package.json                  # Root monorepo
├── README.md
├── docker-compose.yml
│
├── backend/
│   ├── .env.example
│   ├── package.json
│   └── src/
│       ├── server.js             # Express app entry
│       ├── config/
│       │   └── db.js             # MongoDB connection
│       ├── middleware/
│       │   ├── auth.js           # JWT protect / restrictTo
│       │   └── security.js       # helmet, rate limits, sanitizers
│       ├── models/
│       │   ├── User.js
│       │   ├── Coupon.js         # encryptedCode (select:false)
│       │   └── Transaction.js    # full escrow lifecycle
│       ├── controllers/
│       │   ├── auth.js
│       │   ├── coupon.js
│       │   ├── payment.js        # webhook handler
│       │   └── chat.js
│       ├── services/
│       │   ├── escrow.js         # Full escrow pipeline
│       │   ├── juspay.js         # JusPay API wrapper
│       │   └── ai.js             # Anthropic scoring + chatbot
│       ├── routes/
│       │   └── index.js
│       └── utils/
│           ├── encryption.js     # AES-256, OTP, masking
│           ├── logger.js         # Winston
│           └── seed.js           # Demo data
│
└── frontend/
    ├── .env.example
    ├── package.json
    └── src/
        ├── App.js
        ├── index.js
        ├── index.css
        ├── context/
        │   └── AuthContext.js
        ├── services/
        │   └── api.js            # Axios instance + interceptors
        ├── components/
        │   ├── Navbar.js
        │   ├── CouponCard.js
        │   └── ChatWidget.js     # AI support chatbot UI
        └── pages/
            ├── Home.js
            ├── Login.js
            ├── Register.js
            ├── Marketplace.js
            ├── SellCoupon.js
            ├── Payment.js        # OTP entry + coupon reveal
            └── Dashboard.js
```

---

## Quick Start

### Prerequisites
- Node.js ≥ 18
- MongoDB (local or Atlas)
- JusPay sandbox account (free): https://dashboard.juspay.in/register
- Anthropic API key: https://console.anthropic.com

### 1. Clone & Install

```bash
git clone https://github.com/yourname/couponx.git
cd couponx
npm install
npm install --prefix backend
npm install --prefix frontend
```

### 2. Configure Environment

```bash
# Backend
cp backend/.env.example backend/.env
# Edit backend/.env with your keys

# Frontend
cp frontend/.env.example frontend/.env
```

Generate encryption keys:
```bash
# COUPON_ENCRYPTION_KEY (64 hex chars = 32 bytes)
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"

# JWT_SECRET (64+ chars)
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

### 3. Seed Demo Data

```bash
cd backend && npm run seed
```

This creates:
- `buyer@couponx.demo` / `Buyer@123`
- `seller@couponx.demo` / `Seller@123`
- `admin@couponx.demo` / `Admin@123`
- 10 demo coupons with AI scores

### 4. Run Development Servers

```bash
# From root
npm run dev
# Backend: http://localhost:5000
# Frontend: http://localhost:3000
```

### 5. Run with Docker

```bash
docker-compose up --build
```

---

## Environment Variables

### Backend (`backend/.env`)

| Variable | Description |
|----------|-------------|
| `NODE_ENV` | `development` or `production` |
| `PORT` | API port (default: 5000) |
| `MONGODB_URI` | MongoDB connection string |
| `JWT_SECRET` | Min 64 chars random string |
| `JWT_EXPIRE` | e.g. `7d` |
| `JWT_REFRESH_SECRET` | Separate secret for refresh tokens |
| `JWT_REFRESH_EXPIRE` | e.g. `30d` |
| `JUSPAY_MERCHANT_ID` | From JusPay dashboard |
| `JUSPAY_API_KEY` | From JusPay dashboard |
| `JUSPAY_BASE_URL` | `https://sandbox.juspay.in` (sandbox) |
| `JUSPAY_WEBHOOK_SECRET` | For HMAC verification |
| `COUPON_ENCRYPTION_KEY` | 64 hex chars (32 bytes) |
| `ANTHROPIC_API_KEY` | Anthropic Claude API key |
| `FRONTEND_URL` | CORS origin (e.g. `http://localhost:3000`) |
| `OTP_EXPIRY_MINUTES` | OTP validity (default: 10) |

---

## API Reference

### Auth
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/auth/register` | ❌ | Register buyer or seller |
| POST | `/api/auth/login` | ❌ | Login, get tokens |
| POST | `/api/auth/refresh` | ❌ | Refresh access token |
| POST | `/api/auth/logout` | ✅ | Logout |
| GET  | `/api/auth/me` | ✅ | Get current user |

### Coupons
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/coupons` | Optional | List active coupons (masked code only) |
| GET | `/api/coupons/:id` | Optional | Get single coupon |
| POST | `/api/coupons` | Seller | Create + encrypt coupon |
| DELETE | `/api/coupons/:id` | Seller | Remove listing |
| GET | `/api/coupons/my/listings` | Seller | My listings |

### Payments
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/payments/initiate` | Buyer | Start purchase, get JusPay URL |
| POST | `/api/payments/webhook` | JusPay | Payment status updates |
| POST | `/api/payments/verify-otp` | Buyer | Verify OTP → reveal coupon code |
| GET  | `/api/payments/status/:txnId` | Buyer | Poll transaction status |
| GET  | `/api/payments/history` | Buyer | Purchase history |
| GET  | `/api/payments/callback` | ❌ | JusPay redirect handler |

### Chat
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/chat` | Optional | AI support chatbot |

---

## JusPay Integration

1. **Sign up free**: https://dashboard.juspay.in/register
2. Get your `merchant_id` and `api_key` from the sandbox dashboard
3. Set webhook URL in JusPay dashboard: `https://yourdomain.com/api/payments/webhook`
4. Set `JUSPAY_WEBHOOK_SECRET` to verify webhook authenticity

**Test cards (sandbox)**:
- Success: `4111 1111 1111 1111` · Any future date · Any CVV
- Failure: `4000 0000 0000 0002`

---

## AI Features

### Coupon Scoring
- Runs automatically in background after listing
- Scores 0–100 based on: savings %, days left, min order, category demand
- Flags potentially fraudulent listings (high-risk → not published)
- Falls back to heuristic scoring if API unavailable

### Support Chatbot
- Full context of CouponX platform
- Handles: escrow questions, refund queries, OTP issues, how-to questions
- Aware of logged-in user's name and role
- Rate-limited to 30 messages/minute

---

## Deployment

### Backend (Railway / Render / EC2)
```bash
# Set all env vars in your platform
# Build: npm install
# Start: node src/server.js
```

### Frontend (Vercel / Netlify)
```bash
# Build command: npm run build
# Output dir: build
# Set REACT_APP_API_URL=https://yourapi.domain.com/api
```

### Docker Compose (self-hosted)
```bash
docker-compose up -d --build
```

---

## License

MIT — built for demo/educational purposes. Use JusPay sandbox for testing.
