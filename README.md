# QuizBlast — Real-Time Multiplayer Quiz Platform

A free, open-source, **Kahoot-style** real-time multiplayer quiz platform built with **FastAPI** and **React**.

## Features

- 🎮 **Real-time gameplay** via WebSockets
- 👥 **Multiple simultaneous rooms** with complete isolation
- 🏆 **Live leaderboard** after every question
- 🖼️ **Image support** for questions
- ⏱️ **Time-based scoring** — faster answers score more
- 📱 **Mobile-friendly** participant view
- 🔒 **JWT auth** for hosts; participants join with just a nickname + PIN

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | FastAPI (Python 3.11+), SQLAlchemy, Alembic, SQLite |
| Real-time | Native FastAPI WebSockets |
| Frontend | React 18 + Vite, Tailwind CSS, Zustand |
| Auth | JWT (python-jose + passlib) |
| Images | Local filesystem + UUID naming |

---

## Quick Start

### 1. Backend

```bash
cd backend

# Create and activate virtualenv
python -m venv venv
venv\Scripts\activate          # Windows
# source venv/bin/activate     # macOS/Linux

# Install dependencies
pip install -r requirements.txt

# Configure environment
copy .env.example .env         # Windows
# cp .env.example .env         # macOS/Linux
# Edit .env and set a strong SECRET_KEY

# Run database migrations
alembic upgrade head

# Start the backend server
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

API docs: http://localhost:8000/docs

### 2. Frontend

```bash
cd frontend
npm install
npm run dev
```

App: http://localhost:5173

---

## Usage Flow

### As a Host
1. Register/login at `/host/login`
2. Create a quiz with questions at `/host/quiz/new`
3. Click **Start Game** on the dashboard → a PIN is generated
4. Share the PIN with players
5. Click **Start Game** when players have joined
6. Use **Next Question** to advance through the quiz
7. View the final results

### As a Player
1. Go to `/join` and enter the 6-character PIN + a nickname
2. Wait in the lobby for the host to start
3. Answer questions within the time limit
4. See your score and the leaderboard after each question
5. View the final podium at the end

---

## Architecture Decisions

### WebSocket Room Isolation
Every game room is stored in an in-memory `active_rooms` dict keyed by `room_code`. All game events operate only on `active_rooms[room_code]` — no cross-room data access is ever done.

### In-Memory vs Database
- **Database (SQLite/PostgreSQL)**: Stores quiz definitions, question content, user accounts, and room history (for audit)
- **In-Memory**: All live game state (player scores, current question, answer counts) lives in Python dataclasses. This eliminates DB round-trips during gameplay, enabling low-latency real-time interactions.

### WebSocket Protocol
Two separate endpoints:
- `ws://.../ws/host/{room_code}?token={jwt}` — authenticated host
- `ws://.../ws/player/{room_code}` — participants (send `player_join` as first message)

All messages are JSON: `{ "type": "event_name", "payload": { ... } }`

### Scoring
```
Full points for the first 20% of time window.
Linear decrease from 100% → 50% for the remaining 80% of time.
Wrong answers: always 0 points.
```

---

## Swapping SQLite → PostgreSQL

1. Install: `pip install psycopg2-binary`
2. In `.env`, set:
   ```
   DATABASE_URL=postgresql://user:password@localhost:5432/quizdb
   ```
3. Run: `alembic upgrade head`

No code changes needed — SQLAlchemy handles both.

---

## Swapping Local Images → S3 / Cloudinary

In `backend/app/routers/upload.py`, replace the file-write block with an S3 upload:

```python
import boto3
s3 = boto3.client('s3')
s3.put_object(Body=data, Bucket='your-bucket', Key=filename, ContentType=file.content_type)
image_url = f"https://your-bucket.s3.amazonaws.com/{filename}"
```

For Cloudinary, use `cloudinary.uploader.upload(data)`.

---

## Project Structure

```
quiz-platform/
├── backend/
│   ├── main.py              # FastAPI app entry point
│   ├── requirements.txt
│   ├── .env.example
│   ├── alembic/             # Database migrations
│   └── app/
│       ├── config.py        # Settings
│       ├── database.py      # SQLAlchemy engine
│       ├── models/          # DB models (User, Quiz, Question, Room)
│       ├── schemas/         # Pydantic schemas
│       ├── routers/         # REST API + WebSocket endpoints
│       ├── services/        # Business logic
│       ├── websocket/       # WS events, game state, handlers
│       └── utils/           # Scoring algorithm
└── frontend/
    └── src/
        ├── api/             # Axios wrappers
        ├── store/           # Zustand state
        ├── hooks/           # useGameSocket
        ├── components/      # Shared UI components
        └── pages/           # Route pages
```

---

## License

MIT — free to use, modify, and distribute.
