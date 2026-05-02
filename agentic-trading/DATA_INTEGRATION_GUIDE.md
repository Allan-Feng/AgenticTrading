# Data Integration Guide — May 1, 2026 (5:33 PM EDT)

**Working Directory:** `/home/allan/.openclaw/workspace/AgenticTrading/agentic-trading/`

---

## What Changed

### Frontend (`app.js`) Now Fetches Real Data
✅ **Removed:** Mock data generation (`generateCurve()`)
✅ **Added:** API calls to backend endpoints
✅ **Real data sources:**
- `/runs` — All backtest runs
- `/compare` — Comparison data (equity curves, metrics)

---

## How to Use

### 1. Start Backend
```bash
cd /home/allan/.openclaw/workspace/AgenticTrading/agentic-trading
python3 backend/app.py
```

### 2. Open Frontend
```
http://localhost:8000/
```

### 3. Backend Loads Data Automatically
- Page loads → `app.js` fetches `/runs` and `/compare`
- Chart displays real equity curves
- All metrics populated from database

---

## API Endpoints Being Used

| Endpoint | Purpose | Response |
|----------|---------|----------|
| `GET /runs` | List all backtest runs | Array of run metadata |
| `GET /compare` | Get comparison data | Equity curves, timestamps, metrics |

---

## Data Flow

```
Browser (frontend/app.js)
         ↓
    fetch('/runs')
         ↓
   Backend (FastAPI)
         ↓
   SQLite Database
         ↓
    JSON Response
         ↓
Browser (app.js processes → chart displays)
```

---

## What Gets Displayed

### From `/runs`:
- Run IDs, agent names, dates
- Equity metrics (Sharpe, max drawdown, etc.)

### From `/compare`:
- **equity_curves** → Multi-line chart (Agent vs Baselines)
- **timestamps** → X-axis labels
- **metrics** → Performance summary right panel

---

## Chart Details

**Real Data Features:**
- ✅ Multiple agents on same chart
- ✅ Market baselines (SPY, Equal-Weight)
- ✅ Proper timestamps from database
- ✅ Interactive tooltips with real values
- ✅ Responsive to zoom/pan

---

## Troubleshooting

### Chart shows no data?
1. Check backend is running: `python3 backend/app.py`
2. Check database exists: `data/backtest.db`
3. Check browser console for errors: `F12 → Console`
4. Verify endpoint responds: `curl http://localhost:8000/runs`

### Wrong data showing?
1. Make sure backtest has been run
2. Check database isn't empty: `sqlite3 data/backtest.db "SELECT COUNT(*) FROM runs;"`

### API errors?
- CORS: Backend handles CORS headers automatically
- Connection: Ensure backend port 8000 is accessible

---

## Files Modified

| File | Changes |
|------|---------|
| `app.js` | ✅ Removed mock data, added API integration |
| `index.html` | ℹ️ No changes |
| `styles.css` | ℹ️ No changes |

---

## Next Steps (Optional)

1. **Decision Log** → Fetch from `/paper/trades` or decision log endpoint
2. **Live Updates** → Switch `/compare` call to WebSocket for real-time updates
3. **Paper Trading** → Show live account info from `/paper/account`

---

## Testing

1. Start backend
2. Run a backtest (or use existing data in `backtest.db`)
3. Reload frontend
4. Chart should populate with real data

---

**Status:** ✅ Data integration complete  
**Backend:** Ready at http://localhost:8000/  
**Frontend:** Pulling real data from `/runs` and `/compare`
