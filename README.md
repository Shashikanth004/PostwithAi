# 🔥 Omnichannel Orchestration Engine v4

Full-stack social media content engine with **Google Sign-In**, **6-platform OAuth account connections**, **MongoDB credential storage**, **Gemini Vision**, and **per-platform scheduling**.

---

## 🏗 Architecture

```
omnichannel_v4/
├── backend/
│   ├── main.py                    ← FastAPI app entry point
│   ├── requirements.txt
│   ├── .env.example               ← Copy to .env and fill keys
│   ├── core/
│   │   ├── config.py              ← All env var settings
│   │   ├── database.py            ← MongoDB Motor client
│   │   ├── security.py            ← JWT + AES-256 token encryption
│   │   └── deps.py                ← FastAPI auth dependency
│   ├── routers/
│   │   ├── auth.py                ← Google Sign-In OAuth flow
│   │   ├── social_accounts.py     ← Per-platform OAuth connect/disconnect
│   │   └── orchestration.py      ← Gemini content generation + history
│   └── services/                  ← (extend here: post publishing services)
│
└── frontend/
    ├── src/
    │   ├── main.jsx               ← App entry with AuthProvider
    │   ├── App.jsx                ← Router + main orchestration UI
    │   ├── context/
    │   │   └── AuthContext.jsx    ← Google login state + JWT management
    │   ├── pages/
    │   │   ├── LoginPage.jsx      ← Google Sign-In landing
    │   │   ├── AuthCallback.jsx   ← Handles /auth/callback#token=...
    │   │   └── AccountsPage.jsx   ← Connect/disconnect social accounts
    │   ├── components/
    │   │   ├── Header.jsx         ← Nav + user avatar + logout dropdown
    │   │   ├── PlatformSelector.jsx
    │   │   ├── MediaUploader.jsx
    │   │   ├── ScheduleManager.jsx
    │   │   ├── LoadingState.jsx
    │   │   ├── PlatformCard.jsx
    │   │   └── ResultsPanel.jsx
    │   └── utils/
    │       ├── api.js             ← Axios client with JWT injection
    │       └── constants.js
    └── ...
```

---

## 🚀 Setup

### Prerequisites
- Python 3.11+
- Node.js 18+
- MongoDB running locally (`mongod`) or MongoDB Atlas URL

### 1. Install & configure

```bash
# Clone / unzip the project
cd omnichannel_v4

# Backend
cd backend
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
# → Open .env and fill ALL keys (see below)

# Frontend
cd ../frontend
npm install
```

### 2. Start MongoDB

```bash
# macOS (Homebrew)
brew services start mongodb-community

# Ubuntu/Debian
sudo systemctl start mongod

# Or use MongoDB Atlas — paste the connection string in .env MONGODB_URL
```

### 3. Run

```bash
# One command from project root:
./start.sh

# Or manually:
# Terminal 1:
cd backend && source venv/bin/activate && uvicorn main:app --reload --port 8000
# Terminal 2:
cd frontend && npm run dev
```

Open **http://localhost:5173**

---

## 🔑 API Keys Setup

### Google OAuth (app login + YouTube)
1. Go to [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
2. Create **OAuth 2.0 Client ID** → Web application
3. Add Authorized redirect URI: `http://localhost:8000/auth/google/callback`
4. Also enable **YouTube Data API v3** for YouTube posting
5. Paste `client_id` and `client_secret` into `.env`

### LinkedIn
1. [LinkedIn Developers](https://www.linkedin.com/developers/apps) → Create app
2. Auth tab → Add redirect URL: `http://localhost:8000/social/linkedin/callback`
3. Request scopes: `r_liteprofile`, `r_emailaddress`, `w_member_social`

### X (Twitter)
1. [Twitter Developer Portal](https://developer.twitter.com/en/portal/dashboard)
2. Create project + app → Enable OAuth 2.0
3. Callback: `http://localhost:8000/social/x/callback`
4. Scopes: `tweet.read tweet.write users.read offline.access`

### Meta (Instagram + Facebook)
1. [Meta for Developers](https://developers.facebook.com/apps) → Create app
2. Add Facebook Login → redirect: `http://localhost:8000/social/instagram/callback`
3. Scopes: `instagram_basic`, `instagram_content_publish`, `pages_show_list`

### TikTok
1. [TikTok Developers](https://developers.tiktok.com/) → Create app
2. Add redirect: `http://localhost:8000/social/tiktok/callback`
3. Scopes: `user.info.basic`, `video.publish`, `video.upload`

### Pinterest
1. [Pinterest Developers](https://developers.pinterest.com/apps/) → Create app
2. Add redirect: `http://localhost:8000/social/pinterest/callback`

---

## 🔐 Security Model

| Layer | Implementation |
|-------|---------------|
| App auth | Google OAuth 2.0 → JWT (7-day expiry, HS256) |
| Token storage | AES-256 Fernet encryption before MongoDB write |
| Token transmission | Never sent to frontend — only decrypted in server memory at post time |
| State CSRF | Per-request random state token verified on OAuth callback |
| PKCE | Enabled for X (Twitter) as required by their OAuth 2.0 spec |

---

## 🗄 MongoDB Collections

| Collection | Contents |
|-----------|----------|
| `users` | Google profile info, last login |
| `social_accounts` | Connected accounts with **encrypted** access/refresh tokens |
| `scheduled_posts` | Pending scheduled posts |
| `orchestration_history` | Past generation results per user |

---

## 📡 API Endpoints

### Auth
| Method | Path | Description |
|--------|------|-------------|
| GET | `/auth/google/login` | Redirect to Google consent |
| GET | `/auth/google/callback` | Exchange code, issue JWT, redirect to frontend |
| GET | `/auth/me` | Get current user info |

### Social Accounts
| Method | Path | Description |
|--------|------|-------------|
| GET | `/social/{platform}/connect` | Start OAuth for platform |
| GET | `/social/{platform}/callback` | OAuth callback, store encrypted tokens |
| GET | `/social/accounts` | List connected accounts (no tokens exposed) |
| DELETE | `/social/{platform}/disconnect` | Remove account from DB |

### Orchestration
| Method | Path | Description |
|--------|------|-------------|
| POST | `/orchestrate` | Generate platform-native content (multipart) |
| GET | `/history` | Past 20 orchestrations for current user |

---

## 🛠 Extending

**Add direct posting:** In `backend/services/`, create `post_{platform}.py` that fetches the decrypted token via `get_decrypted_token(user_id, platform)` from `social_accounts.py` and calls the platform's posting API.

**Add token refresh:** Extend `social_accounts.py` to detect expired tokens and use the stored `refresh_token` to get a new `access_token` automatically.
