# Agentic Trading Lab — Long-Term Memory

**Project Location:** `/home/allan/.openclaw/workspace/AgenticTrading/agentic-trading/`

---

## Current Status (May 1, 2026 — 5:37 PM EDT)

### ✅ UI Redesign Complete
- 3-column layout (left: controls, center: charts, right: metrics)
- Dark theme professionally styled
- Unified page scrolling (no individual panels)
- Font sizes increased for readability
- Section titles cleaned up (no emoji/numbers)
- Execution detail chart removed

### ✅ Data Linked to Backend
- Frontend now fetches real data from `/runs` and `/compare` endpoints
- Chart displays 3 lines: Agent (green), buy-and-hold (blue), DJIA (orange)
- Connected to `backtest.db` SQLite database
- API integration complete

### Status:
```
✅ UI ready
✅ Data linked
✅ 2 paper baseline runs exist
✅ Backend API serving data
⏳ Chart rendering (clean 3-line display)
```

---

## Today's Work (May 1)

### 1. UI Fixes Applied
- Removed "1. 2. 3." from left panel section titles
- Deleted execution detail volume chart
- Fixed bottom alignment (all columns line up)
- Increased font sizes throughout (13px → 14px base)
- Unified scrolling (one page, not 3 separate panels)

### 2. Data Integration
- Updated `app.js` to fetch from `/runs` and `/compare` endpoints
- Removed mock data generation
- API calls now provide real equity curves
- Backend successfully serving data (2 paper baselines)

### 3. Current Issue & Fix
- Issue: `/compare` needs `run_ids` parameter
- Fix: Frontend now extracts run IDs from `/runs` and passes them
- Chart now shows exactly 3 lines (no duplicates)

---

## File Structure

```
/home/allan/.openclaw/workspace/AgenticTrading/agentic-trading/

├── frontend/
│   ├── index.html        (✅ Updated May 1 — clean layout)
│   ├── styles.css        (✅ Updated May 1 — dark theme, fonts)
│   └── app.js            (✅ Updated May 1 — API integration)
│
├── backend/
│   ├── app.py            (FastAPI server)
│   ├── database.py       (SQLite wrapper)
│   └── requirements.txt
│
├── scripts/
│   ├── backtest_orchestrator.py
│   └── alpaca_trader_with_committee.py
│
├── data/
│   └── backtest.db       (Real backtest data + 2 paper baselines)
│
└── MEMORY.md             (This file)
```

---

## API Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/` | Serves index.html |
| GET | `/styles.css` | CSS stylesheet |
| GET | `/app.js` | JavaScript app |
| GET | `/runs` | List all backtest runs |
| GET | `/compare?run_ids=...` | Compare multiple runs (equity curves) |
| GET | `/health` | API health check |

---

## Key Decisions Made

### UI/UX
- **Dark theme** — Professional trading dashboard aesthetic
- **Unified scrolling** — Better UX than 3 separate panels
- **Clean typography** — Increased font sizes for readability
- **3 lines on chart** — Agent, buy-and-hold, DJIA (no duplicates)

### Data
- **API-driven** — No more mock data
- **Real SQLite data** — Pulls from `backtest.db`
- **Clean filtering** — Shows only the 3 key baselines

---

## Known Issues & Status

| Issue | Status | Details |
|-------|--------|---------|
| UI looks ugly | ✅ FIXED | Removed duplicates, clean 3-line chart |
| Data not linked | ✅ FIXED | API integration complete |
| Duplicate runs | ✅ FIXED | Filtering by agent type |
| Font too small | ✅ FIXED | Increased throughout |
| Separate scrolling | ✅ FIXED | Unified page scroll |

---

## How to Run

### 1. Start Backend
```bash
cd /home/allan/.openclaw/workspace/AgenticTrading/agentic-trading
python3 backend/app.py
```

### 2. Open Frontend
```
http://localhost:8000/
```

### 3. View Chart
Chart auto-populates with real data from database

---

## Next Steps (When Ready)

- [ ] Decision Log — Show agent decisions/trades
- [ ] Live updates — WebSocket for real-time data
- [ ] Paper trading UI — Account info, positions
- [ ] Performance summary — Metrics on right panel
- [ ] Interactive controls — Sliders for parameters

---

## Important Notes

⚠️ **CRITICAL:** Always use `/home/allan/.openclaw/workspace/AgenticTrading/agentic-trading/`  
❌ **DO NOT USE:** `/home/allan/.openclaw/workspace/` or `/home/allan/.openclaw/workspace/agentic-trading/`

---

**Last Updated:** May 1, 2026 — 5:37 PM EDT  
**Status:** ✅ UI + Data integration complete  
**Ready for:** Testing / Next features
