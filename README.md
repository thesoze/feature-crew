# Feature Crew — Autonomous Development Pipeline

A production-tested, reusable framework for autonomous code generation, testing, and review. Originally built for the Argus project, extracted as a standalone template for any Python project.

**Cost:** ~$0.0005 per task (local LLM + cheap API review) vs. $0.15+ with traditional approach.  
**Speed:** 30-60 seconds per task (code generation + self-audit + review).  
**Reliability:** Fresh-context review only (no conversation context bloat).

## Quick Start

```bash
# 1. Clone this repo
git clone https://github.com/your-org/feature-crew.git myproject-crew
cd myproject-crew

# 2. Configure for your project
cp config.example.toml config.toml
# Edit config.toml: set LLM models, Redis URL, project paths

# 3. Start the infrastructure
docker-compose up -d  # Redis + optional Spark service

# 4. Run the pipeline
python -m feature_crew tier1 orchestrator

# 5. Enqueue a task
python -m feature_crew enqueue \
  --spec "Add a function named greet() that returns 'Hello'" \
  --spec-slug "test-greet-function"
```

## Architecture

```
Task Queue (Redis)
    ↓
Spark Service (tier-specific task dispatch)
    ├─ Tier 1 (XS, low-risk): Local LLM (Qwen, Llama, etc.)
    ├─ Tier 2 (S/M, standard): Claude Sonnet
    └─ Tier 3 (L/XL, high-risk): Claude Opus + security review
    ↓
Code Generation (via Aider or direct LLM)
    ↓
Self-Audit (pytest, ruff, mypy, compile)
    ↓
Fresh-Context Review (read diff + audit only, never conversation history)
    ↓
Merge & Deploy (or re-queue with feedback, up to 3 attempts)
```

## Core Components

### 1. Task Router
- **File:** `feature_crew/router.py`
- **Classification:** XS/S/M/L/XL by LOC, complexity, risk
- **Output:** Task payload with `tier`, `model`, `crew_plan`

### 2. Spark Service
- **File:** `spark/tier1_service.py`
- **Responsibility:** Manage task queue, create worktrees, track state
- **Protocol:** REST API (GET /next, POST /complete, POST /feedback)
- **Storage:** Redis queue + task metadata

### 3. Orchestrator
- **File:** `feature_crew/orchestrator.py`
- **Tiers:** Pluggable orchestrator per tier
  - `Tier1Orchestrator` — local LLM dispatch (Qwen, Llama)
  - `Tier2Orchestrator` — Claude Sonnet (via API)
  - `Tier3Orchestrator` — Claude Opus + security review
- **Method:** Poll Spark service, dispatch, wait for audit, review, merge

### 4. Self-Audit
- **File:** `feature_crew/audit/`
- **Tools:** pytest, ruff, mypy, py_compile (configurable)
- **Verdict:** `ready_for_review` | `needs_retry`
- **Output:** JSON report with all check results

### 5. Fresh-Context Review
- **File:** `feature_crew/review.py`
- **Reads:** spec + diff + audit report ONLY
- **Does NOT read:** conversation history, task metadata, prior attempts
- **Checks:** Custom validation chain (5+ checks per tier)
- **Output:** `{action: approve|reject, reason, feedback}`

### 6. CLI
- **Commands:**
  - `enqueue` — add task to queue
  - `tier1 orchestrator` — start tier-1 polling loop
  - `tier2 orchestrator` — start tier-2 polling loop
  - `status` — check queue status
  - `config` — show/test configuration

## Configuration

### config.toml

```toml
[project]
name = "myproject"
repo_path = "/path/to/myproject"
description = "My awesome project"

[infrastructure]
redis_url = "redis://localhost:6379"
spark_url = "http://localhost:30001"

[tier1]
enabled = true
model = "qwen3-235b"  # or "llama3-70b", "mistral-7b", etc.
guardrail_file = "guardrails/tier1-f2.md"  # optional: teach-to-ask guardrail
timeout_seconds = 300

[tier2]
enabled = true
model = "claude-sonnet-4-6"  # Claude API
api_key = "sk-..."  # or read from .env
cost_budget = 5.0  # USD per day

[tier3]
enabled = false  # tier-3 for high-risk only
model = "claude-opus-4-7"
security_review = true

[self_audit]
tools = ["pytest", "ruff", "mypy", "compile"]
pytest_args = ["tests/", "-v", "--tb=short"]
ruff_args = ["check", "src/", "--select=E9,F63,F7,F82"]

[review]
# Validation checks (ordered, stop on first failure)
checks = [
  "audit_verdict",
  "diff_size",
  "file_count",
  "spec_adherence",
  "suspicious_patterns",
]
max_files_per_tier1 = 3
max_files_per_tier2 = 10
```

### guardrails/tier1-f2.md

Optional guardrail to teach tier-1 LLM to ask on ambiguous specs:

```markdown
# Tier-1 F2 Guardrail

When given a vague spec, respond with:
[ASK]: <clarification request>

Vague spec indicators:
- "improve", "clean up", "refactor" (without specifics)
- "make it better", "optimize", "simplify"
- "handle edge cases" (which ones?)

Example:
Spec: "Improve the login flow"
Response: [ASK]: Do you want to (a) add rate limiting, (b) fix a specific bug, (c) add 2FA?
```

## Usage Patterns

### Pattern 1: Quick Tier-1 (Development)

```bash
# Single command for dev/test features
python -m feature_crew enqueue \
  --spec "Add validation to email field in UserForm" \
  --spec-slug "validate-email-form"

# Orchestrator auto-routes to tier-1 (Qwen3), audits locally, reviews cheaply
# Cost: ~$0.0005, Time: ~40s
```

### Pattern 2: Tier-2 (Standard Feature)

```bash
# For more complex features (multi-file, new endpoint, schema changes)
python -m feature_crew enqueue \
  --spec "Add OAuth2 login endpoint with database migration" \
  --spec-slug "oauth2-login" \
  --tier 2

# Routes to Sonnet, fuller review, costs ~$0.15
# Time: ~2-3 minutes
```

### Pattern 3: Tier-3 (High-Risk)

```bash
# For security-sensitive, auth, payment, external IO
python -m feature_crew enqueue \
  --spec "Implement HMAC verification for webhook signatures" \
  --spec-slug "webhook-hmac" \
  --tier 3

# Routes to Opus + fresh security review, full suite, costs ~$0.30
# Time: ~5-10 minutes
```

## Extensibility

### Add a New LLM Model

1. **Create tier orchestrator:**
   ```python
   # feature_crew/orchestrators/mistral_tier1.py
   class MistralTier1Orchestrator(BaseTier1Orchestrator):
       async def dispatch_to_llm(self, spec, worktree_path, feedback):
           # Call Mistral API or local vLLM endpoint
           pass
   ```

2. **Register in config:**
   ```toml
   [tier1]
   model = "mistral-7b"
   api_endpoint = "http://localhost:8000"
   ```

3. **Optional guardrail:**
   ```markdown
   # guardrails/mistral-tier1.md
   # Custom guardrail for Mistral's quirks
   ```

### Add Self-Audit Tool

1. **Create audit plugin:**
   ```python
   # feature_crew/audit/plugins/bandit.py
   def run_bandit(task_id, worktree_path, files_modified):
       result = subprocess.run(["bandit", "-r", "src/"])
       return {"exit_code": result.returncode, "passed": result.returncode == 0}
   ```

2. **Register in config:**
   ```toml
   [self_audit]
   tools = ["pytest", "ruff", "mypy", "bandit"]
   ```

### Add Review Check

1. **Create custom validator:**
   ```python
   # feature_crew/review/checks/security.py
   def check_no_hardcoded_secrets(spec, diff, audit_report):
       if "password" in diff or "secret" in diff:
           return False, "Found hardcoded secrets in diff"
       return True, None
   ```

2. **Register:**
   ```python
   # feature_crew/review.py
   CHECKS = [
       check_audit_verdict,
       check_diff_size,
       check_no_hardcoded_secrets,  # custom
   ]
   ```

## Deployment

### Local Development

```bash
# Terminal 1: Redis
docker run -d -p 6379:6379 redis:latest

# Terminal 2: Spark service
python spark/tier1_service.py

# Terminal 3: Orchestrator
python -m feature_crew tier1 orchestrator
```

### Production

```bash
# Using docker-compose
docker-compose -f docker-compose.prod.yml up -d

# Using systemd (Linux)
sudo systemctl start feature-crew-orchestrator.service

# Using launchd (macOS)
launchctl load ~/Library/LaunchAgents/com.feature-crew.plist
```

## Monitoring

### Health Checks

```bash
# Spark service health
curl http://localhost:30001/health

# Queue status
python -m feature_crew status

# Logs
tail -f logs/orchestrator.log
tail -f logs/spark.log
```

### Metrics

```bash
# Task success rate
redis-cli keys "task:*:status" | wc -l

# Cost tracking
python -m feature_crew metrics --duration 7d

# Latency per tier
python -m feature_crew metrics --tier 1 --metric latency
```

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Task hangs in "dispatch" | LLM endpoint unreachable | Check vLLM/API endpoint URL in config |
| Self-audit fails silently | Tool not installed | `pip install pytest ruff mypy` |
| Review rejects all diffs | Too-strict checks | Adjust thresholds in config + review.py |
| Merge conflicts | main branch has new commits | Implement smart merge (rebase + retry) |

## Project Structure

```
feature-crew/
├── feature_crew/
│   ├── __init__.py
│   ├── cli.py               # CLI commands
│   ├── router.py            # Task triage (tier classification)
│   ├── orchestrators/       # Pluggable per-tier orchestrators
│   │   ├── base.py
│   │   ├── tier1_qwen.py
│   │   ├── tier1_mistral.py
│   │   ├── tier2_sonnet.py
│   │   └── tier3_opus.py
│   ├── audit/               # Self-audit framework
│   │   ├── runner.py
│   │   └── plugins/
│   │       ├── pytest.py
│   │       ├── ruff.py
│   │       ├── mypy.py
│   │       └── compile.py
│   ├── review/              # Fresh-context review
│   │   ├── checks.py
│   │   └── validators/
│   ├── models/              # Data classes (Task, AuditReport, etc.)
│   └── config.py            # Configuration loader
├── spark/                   # Spark service (tier-specific dispatch)
│   ├── tier1_service.py
│   ├── requirements.txt
│   └── README.md
├── guardrails/              # LLM guardrails (optional)
│   ├── tier1-f2.md
│   └── tier2-instructions.md
├── scripts/
│   ├── enqueue.py
│   ├── status.py
│   └── metrics.py
├── tests/
│   ├── test_router.py
│   ├── test_orchestrator.py
│   ├── test_audit.py
│   └── test_review.py
├── docker-compose.yml       # Local dev
├── docker-compose.prod.yml  # Production
├── config.example.toml      # Template config
├── pyproject.toml           # Python dependencies
└── README.md                # This file
```

## Contributing

Feature Crew is designed to be extended for your specific project:

1. **Fork or clone** this repo
2. **Customize** config.toml, guardrails/, and review checks
3. **Test** with a few tier-1 tasks before committing to tier-2/3
4. **Monitor** success rate, cost, latency
5. **Iterate** — gather failures, refine guardrails, adjust checks

## License

MIT — use freely in your projects.

## Credits

Built and battle-tested in the **Argus** project (always-on personal AI agent with 35+ OSINT feeds). Extracted as a reusable template to bring the same autonomous development workflow to any Python project.

## Support

- **Issues:** File an issue on GitHub
- **Docs:** Read `docs/` folder for deeper architecture
- **Examples:** See `examples/` for project-specific configs
