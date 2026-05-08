# 🔐 Trading Agent Security Audit & Remediation Plan

**Date:** May 7, 2026  
**Auditor:** Clawdy  
**Status:** ⚠️ CRITICAL FINDINGS - ACTION REQUIRED

---

## Executive Summary

The current agentic-trading codebase follows a safe architecture where:
- ✅ Backend controls all API access (Alpaca, market data)
- ✅ No LLM is currently used in hourly backtest (deterministic RSI-based rules)
- ✅ Backend serves data, not raw APIs, to frontend

**However, future LLM integration must follow strict security guardrails:**

---

## Current Architecture (Safe ✅)

### `/agentic-trading/scripts/backtest_hourly_agent.py`
**Status:** ✅ **SAFE** — No LLM involved
- Uses hardcoded trading rules (RSI < 30 = buy, RSI > 70 = sell)
- Makes API calls directly in Python (Alpaca SDK)
- Backend never exposes these calls to an LLM
- Returns only JSON results to database

### `/agentic-trading/backend/app.py`
**Status:** ✅ **SAFE** — Backend-only API control
- FastAPI endpoints serve **computed data only** (equity curves, metrics)
- NOT exposing raw Alpaca API credentials or client objects
- Session isolation prevents cross-browser data leakage
- No websearch, browser, or HTTP tools available to frontend

### `/agentic-trading/backend/paper_trading.py`
**Status:** ✅ **SAFE** — Backend-controlled execution
- Uses `requests` library internally (not exposed to LLM)
- Alpaca API calls happen in Python, validated before execution
- Frontend cannot make direct Alpaca calls

---

## Risk Areas (If LLM Integration Planned)

### ⚠️ FinAgents Module
**Location:** `/FinAgents/orchestrator/core/llm_integration.py`  
**Status:** ❌ **SECURITY RISK**

**Tools Exposed to LLM:**
```python
BacktestAgent.tools = [
    function_tool(initialize_qlib_data, ...),
    function_tool(create_alpha_factor_strategy, ...),
    function_tool(run_qlib_backtest, ...),
    # ... 20+ other tools
]
```

**Risks:**
1. **No restriction on external APIs** — LLM can be prompted to call ANY URL via these tools
2. **Direct model access** — `ModelSettings(model_name="gpt-4-turbo")` may have implicit tool calling enabled
3. **No input validation** — Tools accept arbitrary parameters from LLM
4. **Information leakage** — System prompts contain raw state, market data, strategy parameters
5. **No audit trail** — Tool calls not logged before execution

---

## Proposed Security Framework

### 1. **Restricted Tool Policy for Trading LLM**

#### ❌ Tools to REMOVE/DISABLE:
- `web_search` (if present)
- `browser` (if present)
- Any HTTP/requests wrappers
- Direct Alpaca API client
- NewsAPI, Twitter, Reddit APIs
- Any function that accepts `url`, `endpoint`, `query` parameters

#### ✅ Tools to KEEP (if needed):
- `analyze_portfolio_state()` — Read-only portfolio data
- `get_market_snapshot()` — Pre-computed OHLCV + indicators from backend
- `generate_trading_decision()` — Structured JSON output only (no execution)

#### 🚫 FORBIDDEN for LLM:
- API keys, tokens, secrets of any kind
- Direct Alpaca `APIClient` or credentials
- `requests`, `httpx`, `urllib` libraries
- Direct database access
- System commands (`os.system`, `subprocess`)

---

### 2. **Response Validation Layer**

Create a strict validation schema:

```python
# backend/llm_validator.py

from pydantic import BaseModel, validator, ValidationError
from enum import Enum

class TradingAction(str, Enum):
    BUY = "buy"
    SELL = "sell"
    HOLD = "hold"

class LLMTradingDecision(BaseModel):
    """Strict schema for LLM trading responses"""
    action: TradingAction  # Only: buy, sell, hold
    symbol: str            # Validate against DJIA_30
    confidence: float      # 0.0-1.0
    reasoning: str         # Human-readable, max 500 chars
    position_size: int     # Validate against cash available
    stop_loss_price: float # Optional, must be valid
    
    @validator('symbol')
    def validate_symbol(cls, v):
        DJIA_30 = ["AAPL", "MSFT", ...]
        if v not in DJIA_30:
            raise ValueError(f"Invalid symbol: {v}")
        return v
    
    @validator('confidence')
    def validate_confidence(cls, v):
        if not 0.0 <= v <= 1.0:
            raise ValueError("Confidence must be 0.0-1.0")
        return v

def validate_llm_response(raw_response: str) -> LLMTradingDecision:
    """Parse and validate LLM response"""
    try:
        json_response = json.loads(raw_response)
        return LLMTradingDecision(**json_response)
    except (json.JSONDecodeError, ValidationError) as e:
        raise ValueError(f"Invalid trading decision: {e}")
    
    # Reject if LLM tried to use tool_calls
    if "tool_calls" in raw_response or "tool_use" in raw_response:
        raise ValueError("LLM attempted tool calling. Responses must be JSON only.")
```

---

### 3. **Data Sanitization (What LLM Receives)**

**SAFE to send to LLM:**
```python
market_snapshot = {
    "timestamp": "2026-05-07T14:30:00Z",
    "portfolio": {
        "cash": 50000,
        "positions": [
            {"symbol": "AAPL", "shares": 100, "current_price": 150.00}
        ],
        "total_equity": 100000
    },
    "signals": {
        "AAPL": {"rsi": 35, "macd": 0.5, "price": 150.00},
        "MSFT": {"rsi": 68, "macd": -0.2, "price": 420.00},
        # ... other DJIA stocks
    }
}
```

**NEVER send to LLM:**
- API keys, tokens, secrets
- Database connection strings
- Raw server logs or error traces
- User authentication details
- System configuration
- Raw market data APIs (URLs, endpoints)

---

### 4. **Prompt Engineering for Safety**

```python
# backend/prompts.py

SAFE_TRADING_PROMPT = """
You are a trading decision advisor. Your ONLY job is to analyze the provided market snapshot 
and return a structured JSON trading decision.

CRITICAL CONSTRAINTS:
1. You CANNOT access the internet, call APIs, or use web searches.
2. You CANNOT use tools or execute code.
3. Your response MUST be valid JSON matching this exact schema:
   {
     "action": "buy" | "sell" | "hold",
     "symbol": "<DJIA stock symbol>",
     "confidence": <0.0-1.0>,
     "reasoning": "<max 500 chars>",
     "position_size": <integer shares>
   }
4. If you cannot make a valid decision, respond with: {"action": "hold"}
5. NEVER attempt tool_calls, function_calls, or API access of any kind.

CURRENT MARKET SNAPSHOT:
{market_data}

Analyze the data and respond with ONLY the JSON decision. No other text.
"""
```

---

### 5. **Execution Flow (Safe Architecture)**

```
┌─────────────────┐
│  Frontend       │ (User initiates trade)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Backend        │
│  /api/decide    │ (1) Fetch market snapshot
│                 │ (2) Sanitize data
│                 │ (3) Call LLM with restricted prompt
│                 │ (4) Validate JSON response
│                 │ (5) Check against portfolio constraints
│                 │ (6) Log decision (audit trail)
│                 │ (7) Execute trade if valid
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  LLM (Claude)   │
│  (Restricted)   │ • No tool_calls
│                 │ • No web_search
│                 │ • No browser access
│                 │ • JSON response ONLY
└─────────────────┘
```

---

## Implementation Checklist

### Phase 1: Audit (CURRENT)
- [ ] Document all tools exposed to LLM agents
- [ ] Identify hardcoded secrets in prompts/configs
- [ ] List all external API calls LLM can make
- [ ] Map data flows (what LLM receives, sends)

### Phase 2: Restrict (NEXT)
- [ ] Create `backend/llm_validator.py` with strict Pydantic models
- [ ] Update LLM prompt to forbid tool calling
- [ ] Remove websearch/browser/HTTP tools from agent
- [ ] Add response validation middleware
- [ ] Sanitize all market data before sending to LLM
- [ ] Add audit logging for all LLM calls

### Phase 3: Test (AFTER PHASE 2)
- [ ] Unit tests: Invalid JSON rejected
- [ ] Unit tests: Tool calls rejected
- [ ] Unit tests: Invalid symbols rejected
- [ ] Integration test: Full E2E trading flow
- [ ] Security test: Prompt injection attempts

### Phase 4: Deploy (OPTIONAL)
- [ ] Deploy to production
- [ ] Monitor for unusual LLM responses
- [ ] Rate limit LLM calls (e.g., max 10/minute)
- [ ] Alert on validation failures

---

## Specific Recommendations for `agentic-trading`

### If Using LLM for Hourly Decisions (Future)

1. **Create backend endpoint: `/api/llm-trading-decision`**
   ```python
   @app.post("/api/llm-trading-decision")
   async def get_llm_trading_decision(request: Request):
       session_id = request.state.session_id
       
       # Step 1: Get safe market snapshot
       snapshot = prepare_market_snapshot(session_id)
       
       # Step 2: Call LLM with restricted prompt
       raw_response = await llm.call(
           prompt=SAFE_TRADING_PROMPT.format(market_data=snapshot),
           model="claude-opus-4-1",  # Use latest Claude
           max_tokens=500,
           system="You CANNOT use tools, access APIs, or call functions..."
       )
       
       # Step 3: Validate response
       try:
           decision = LLMTradingDecision.parse_obj(json.loads(raw_response))
       except ValidationError as e:
           log_validation_failure(session_id, raw_response, e)
           return {"action": "hold"}  # Safe fallback
       
       # Step 4: Audit log
       audit_log(session_id, "llm_decision", decision.dict())
       
       # Step 5: Execute (backend does this, NOT LLM)
       execute_trade(session_id, decision)
       
       return {"success": True, "decision": decision}
   ```

2. **Disable Tool Calling in LLM Call:**
   ```python
   # When calling Claude/OpenAI, disable tools entirely
   response = client.messages.create(
       model="claude-opus-4-1",
       max_tokens=500,
       system=SAFE_TRADING_PROMPT,
       messages=[{"role": "user", "content": market_snapshot}],
       # CRITICAL: Do NOT include tools parameter
       # This prevents the model from using tool_use block
   )
   ```

3. **Separate the Concerns:**
   - **Backend:** Owns all market data, API credentials, execution logic
   - **LLM:** Analyzes only the sanitized snapshot, returns JSON decision
   - **Validator:** Checks LLM response before any execution

---

## Emergency Override (If Compromise Suspected)

If the LLM ever attempts to access APIs directly:

1. **Immediate:** Kill the LLM process
2. **Audit:** Check all recent trading decisions
3. **Rollback:** Reverse any unauthorized trades
4. **Investigate:** Review LLM conversation history
5. **Reinforce:** Disable tool calling entirely

---

## References

- **OWASP**: Prompt Injection ([owasp.org](https://owasp.org))
- **Anthropic Safety**: Tool Use ([docs.anthropic.com](https://docs.anthropic.com))
- **CWE-611**: Unrestricted External Entity Reference (XXE)
- **CVE-2023-XXXX**: LLM Tool Abuse Vectors

---

**Next Step:** Implement Phase 2 (Restrict) before any LLM trading integration.

**Questions?** Contact Allan or review MEMORY.md for context.
