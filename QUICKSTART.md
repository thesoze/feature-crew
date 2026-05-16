# Feature Crew — Quick Start (5 Minutes)

## 1. Clone & Setup

```bash
git clone https://github.com/your-org/feature-crew.git myproject-crew
cd myproject-crew

# Copy example config and customize
cp config.example.toml config.toml
# Edit config.toml: set repo_path, Redis URL, model preferences
```

## 2. Start Infrastructure

```bash
# Option A: Docker (recommended for first time)
docker-compose up -d

# Option B: Manual setup
redis-server
UPSTASH_REDIS_URL=redis://localhost:6379 python spark/tier1_service.py
python -m feature_crew tier1 orchestrator
```

## 3. Test with Tier-1 Task (30 seconds)

```bash
# Enqueue a simple task
python -m feature_crew enqueue \
  --spec "Write a function named greet(name) that returns 'Hello, {name}!'" \
  --spec-slug "test-greet-function"

# Watch the orchestrator logs for:
# 📥 Task received
# 🤖 Dispatching to Qwen3
# ✅ Aider completed
# 📊 Self-audit running
# 🔍 Reviewing...
# ✅ APPROVED (or ❌ REJECTED with reason)
```

## 4. Configuration

### Pick Your LLM Model

**Tier-1 (Autonomous, Local):**
```toml
[tier1]
model = "qwen3-235b"    # or llama3-70b, mistral-7b
vllm_endpoint = "http://localhost:8000/v1"
```

**Tier-2 (Standard, API):**
```toml
[tier2]
model = "claude-sonnet-4-6"
api_key = "${ANTHROPIC_API_KEY}"
```

**Tier-3 (High-Risk, Full Review):**
```toml
[tier3]
enabled = true
model = "claude-opus-4-7"
security_review = true
```

### Customize for Your Project

1. **Guardrails** (teach LLM to ask on ambiguity):
   ```bash
   # Edit guardrails/tier1-f2.md
   # Add project-specific rules, patterns, anti-patterns
   ```

2. **Self-Audit Tools**:
   ```toml
   [tier1.self_audit]
   tools = ["pytest", "ruff", "mypy", "bandit"]
   pytest_args = ["tests/unit", "-v"]
   ```

3. **Review Checks**:
   Edit `feature_crew/review/checks.py` to add custom validators

## 5. Run Your First Real Task

```bash
# Pick a low-risk backlog item (typo fix, simple function, refactor)
python -m feature_crew enqueue \
  --spec "Add validation to email field in UserForm. Must reject invalid emails." \
  --spec-slug "validate-email-userform"

# Cost: ~$0.0005, Time: ~40 seconds
# Result: Code in main, tests passing, auto-merged
```

## 6. Monitor & Iterate

```bash
# Check queue status
python -m feature_crew status

# View task history
python -m feature_crew history --limit 10

# Check success rate
python -m feature_crew metrics --metric success_rate

# View task logs
tail -f logs/feature-crew.log
```

## Next Steps

- **Docs:** Read `docs/architecture.md` for deep dive
- **Customization:** Update config.toml, guardrails/, review checks
- **Scale:** Add more tier-1 tasks, gradually move to tier-2/3
- **Monitor:** Set up Grafana dashboard for cost/latency tracking

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "Task hangs in dispatch" | Check vLLM/API endpoint is reachable |
| "Self-audit fails" | Ensure pytest, ruff, mypy are installed |
| "Review always rejects" | Adjust validation checks in config.toml |
| "Can't reach Redis" | Check Redis is running and URL is correct |

---

**Ready to go.** Start with 5-10 tier-1 tasks, learn the flow, then scale up.
