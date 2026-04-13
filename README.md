# LLM Anonymization

Transparent anonymization proxy for Claude Code in penetration testing engagements.

Sits between Claude Code and the Anthropic API. Every message, bash output, file read, and grep result is anonymized before leaving your machine. Responses are deanonymized before Claude Code sees them. Claude never touches real client data.

---

## How It Works in Practice

You use Claude Code exactly as you normally would — the proxy is invisible. All sensitive data is stripped before it reaches Anthropic and restored before Claude Code sees the response.

```bash
# Claude Code runs nmap and gets real output back
$ claude
> run nmap -sV -sC against 10.20.0.0/24 and tell me what you find

# What Claude actually sees (surrogates):
#   "Nmap scan report for srv-0042.pentest.local (203.0.113.12)"
#   "OpenSSH 8.2 running on srv-0042.pentest.local"

# What you see in your terminal (real data restored):
#   "Nmap scan report for dc01.acmecorp.local (10.20.0.10)"
#   "OpenSSH 8.2 running on dc01.acmecorp.local"
```

Claude reasons about the surrogates and its answers come back with surrogates too — the proxy deanonymizes them before they reach your terminal. **Claude never knows the real target name, IPs, or credentials.**

### What stays protected

- Every bash command output (nmap, crackmapexec, mimikatz, etc.)
- Every file read by Claude Code
- Every grep result, every log snippet
- Credentials you paste into the conversation
- Hostnames, usernames, org names you type directly

### What you still need to handle

- Files you share outside the Claude Code session (reports, notes)
- Screenshots
- Data you copy-paste into other tools

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        YOUR MACHINE                             │
│                                                                 │
│  Claude Code                                                    │
│      │  ANTHROPIC_BASE_URL=http://localhost:8080               │
│      ▼                                                          │
│  LLM Anonymization (FastAPI :8080)                              │
│      │                                                          │
│      ├─ Layer 1: LLM Detector (Ollama qwen3:4b)                │
│      │   └─ Understands context: hostnames, usernames,         │
│      │      org names, credentials, internal system names      │
│      │                                                          │
│      ├─ Layer 2: Regex Safety Net                               │
│      │   └─ Deterministic: IPs, CIDRs, hashes, MACs,           │
│      │      emails, domains, tokens, AWS keys, JWTs            │
│      │                                                          │
│      ├─ PII Vault (SQLite)                                      │
│      │   └─ Persistent surrogate mappings per engagement        │
│      │      original ←→ surrogate, isolated by client          │
│      │                                                          │
│      ▼  [only surrogates leave the machine]                     │
│                                                                 │
└──────────────────────────────────┬──────────────────────────────┘
                                   │
                                   ▼
                         Anthropic API (Claude)
                         sees only fake data
```

### What gets anonymized

| Type | Example | Detected by |
|------|---------|-------------|
| IPv4 / IPv6 | `10.10.50.5`, `fe80::1` | Regex |
| CIDR ranges | `10.10.0.0/16` | Regex |
| Hashes (MD5/SHA1/SHA256/NTLM) | `8846f7eaee8fb117...` | Regex |
| MAC addresses | `aa:bb:cc:dd:ee:ff` | Regex |
| Email addresses | `john@contoso.com` | Regex |
| Domains / FQDNs | `dc01.contoso.local` | Regex |
| URLs | `https://intranet.contoso.com` | Regex |
| AWS / cloud tokens | `AKIAIOSFODNN7EXAMPLE` | Regex |
| JWTs, API keys, session tokens | `eyJhbGci...`, `sk_live_...` | Regex |
| Bare hostnames | `DC01`, `FILESERVER-PRD` | LLM |
| Domain usernames | `CONTOSO\jsmith` | LLM |
| Cleartext passwords | `C0nt0s0@2024!` | LLM |
| Organization names | `Contoso Corporation` | LLM |
| Person names | `John Smith` | LLM |
| Internal app/project names | `Project Phoenix` | LLM |
| Sensitive file paths | `/home/jsmith/docs` | LLM |

### Surrogate format

Surrogates are realistic-looking but clearly non-routable:

| Type | Original | Surrogate |
|------|---------|-----------|
| IP | `192.168.1.10` | `203.0.113.47` (RFC 5737 TEST-NET) |
| Domain | `contoso.local` | `xkqpzt.pentest.local` |
| Hostname | `DC01` | `dc-0042` |
| Username | `john.smith` | `user_rfkw` |
| Email | `john@contoso.com` | `rfkwma@example.pentest` |
| Hash | `8846f7ee...` (32 chars) | random 32-char hex |
| Credential | `C0nt0s0@2024!` | `[CRED_XK9A2B3C]` |

Mappings persist across sessions in SQLite. The same original always maps to the same surrogate within an engagement.

---

## Quick Start

### Option A (primary): VPS — proxy + Ollama in the cloud

The standard production setup: FastAPI proxy and Ollama run on a remote VPS.
An SSH tunnel exposes both on localhost. Nothing needs to be installed locally beyond Python.

```bash
# Terminal 1 — open SSH tunnel (proxy :8080 + Ollama :11434)
VPS_PASS=<password> ./scripts/connect-vps.sh root@<vps-ip> <client-name>
# Keep this running. Ctrl+C closes the tunnel.

# Terminal 2 — Claude Code pointing at the proxy on the VPS
export ANTHROPIC_BASE_URL=http://localhost:8080/<PROXY_SECRET>
export ENGAGEMENT_ID=<client-name>-<date>
claude

# Terminal 3 (optional) — continuous prompt self-improvement loop via VPS Ollama
OLLAMA_HOST=http://localhost:11434 OLLAMA_MODEL=qwen3:4b \
  python -m scripts.feedback_loop --from-failures --auto-apply --continuous
```

> `PROXY_SECRET` is stored in `.env.vps`. Ollama is available at `localhost:11434` while the tunnel is open.

### Option B: Native Python + local Ollama (Apple Silicon)

```bash
# 1. Setup
./scripts/setup.sh
ollama pull qwen3:1.7b   # ~1GB, one-time

# 2. Proxy (terminal 1)
ENGAGEMENT_ID=client-acme-2026 ./scripts/run.sh

# 3. Claude Code (terminal 2)
export ANTHROPIC_BASE_URL=http://localhost:8080
export ENGAGEMENT_ID=client-acme-2026
claude
```

### Option C: Full Docker (everything containerized, CPU only)

```bash
ENGAGEMENT_ID=client-acme-2026 make docker-up
export ANTHROPIC_BASE_URL=http://localhost:8080
claude
```

---

## Engagement Management

**Critical: set a unique ENGAGEMENT_ID per client.** This isolates surrogate mappings so the same IP at two different clients maps to different surrogates.

```bash
# Generate a new engagement ID
./scripts/new-engagement.sh acme

# Check vault stats
curl -s http://localhost:8080/health | python3 -m json.tool

# Clear vault between engagements
ENGAGEMENT_ID=client-acme-2026 make vault-clear
```

---

## Running Tests

```bash
# Unit + regex tests (no Ollama required, completes in seconds)
make test

# Full pipeline tests including LLM
# Requires Ollama accessible — either local or via VPS tunnel (./scripts/connect-vps.sh)
OLLAMA_HOST=http://localhost:11434 OLLAMA_MODEL=qwen3:4b make test-integration
```

The integration test suite enforces a **0% leak policy**: for every pentest fixture (nmap, mimikatz, CrackMapExec, Burp, enum4linux, bash history, LDAP dump, Metasploit), none of the strings in `must_anonymize` may appear in the anonymized output.

---

## Self-Improvement Loop

The anonymization coverage improves over time through a structured loop: add new pentest scenarios, measure what leaks, fix the gaps.

### How it works

```
1. Add a new pentest scenario fixture to tests/fixtures.py
   (new tool, new attack chain, new output format)
        │
        ▼
2. python3 -m scripts.auto_improve --cycles N          ← no Ollama needed
   - Runs ALL fixtures with LLM mocked to empty (regex only)
   - Reports catch rate, leaked values, false positives
   - Classifies leaks: hash, fqdn, credential, hostname, etc.
   - Applies safe auto-fixes (new regex patterns, _NEVER_ANONYMIZE entries)
        │
        ▼
3. Fix remaining regex leaks manually
   - Add targeted patterns to src/regex_detector.py
   - Add known-safe tool names to _NEVER_ANONYMIZE in src/anonymizer.py
   - Re-run auto_improve until 100% catch rate, 0 false positives
        │
        ▼
4. Run integration tests with Ollama        ← Ollama required for this step
   make test-integration
   - Tests the full pipeline: LLM + regex together
   - Any entity the regex missed must be caught by the LLM prompt
   - If the LLM misses something, refine data/system_prompt.txt
     (add the entity type, add negative examples, tighten instructions)
        │
        └─── repeat from step 1 with a new scenario
```

The regex improvement cycle (`auto_improve.py`) requires no Ollama and completes in under 5 seconds.
The LLM prompt improvement step requires Ollama running — either locally or via VPS tunnel.

### Coverage progression (regex layer)

| Fixtures | Items | Catch rate | Scenarios added |
|----------|-------|------------|-----------------|
| 16 | ~160 | ~85% | Initial regex patterns |
| 16 | ~160 | ~97% | LLM layer added |
| 37 | 495 | 100% | Kerberos, NTLM, AWS keys, AD CS, cloud tokens |
| 46 | 610 | 100% | Empire C2, Pacu, Volatility, GoPhish, Shodan, CloudTrail |
| 49 | 645 | 100% | CrackMapExec SMB, Burp Suite HTTP history, Zeek conn.log |

Each new scenario exposes gaps. Fixes from one scenario reliably improve coverage on others — domain\\user patterns added for CrackMapExec also improve Responder and NTDS fixtures.

### Adding a new fixture

```python
# tests/fixtures.py
MY_SCENARIO = PentestFixture(
    name="my_tool_output",
    description="One sentence describing the scenario",
    text="""\
<paste realistic tool output here — use fictional IPs, hostnames, usernames>
""",
    must_anonymize=[
        "192.168.1.10",       # IPs
        "victim.corp",        # domain
        "john.smith",         # username
        "SuperSecret123!",    # credential
    ],
    safe_to_keep=[
        "nmap", "smb", "http", "443",   # tool names, protocols, ports
    ],
)

# Add to ALL_FIXTURES at the bottom of the file
```

Then run:

```bash
python3 -m scripts.auto_improve --cycles 3
```

---

## Configuration

All settings via environment variables or `.env` file (copy from `.env.example`):

| Variable | Default | Description |
|----------|---------|-------------|
| `ENGAGEMENT_ID` | `default` | Isolates vault per client — **change per engagement** |
| `OLLAMA_HOST` | `http://localhost:11434` | Ollama endpoint (local or VPS tunnel) |
| `OLLAMA_MODEL` | `qwen3:1.7b` | Use `qwen3:4b` for better quality |
| `LLM_ENABLED` | `true` | Set `false` to run regex-only (faster, lower coverage) |
| `OLLAMA_TIMEOUT` | `30` | Seconds before giving up on LLM (falls back to regex) |
| `LLM_CHUNK_SIZE` | `1500` | Characters per chunk for long tool outputs |
| `PORT` | `8080` | Proxy listen port |

---

## LLM Model Selection

| Model | Quality | Speed (CPU) | Use when |
|-------|---------|-------------|----------|
| `qwen3:0.6b` | Basic | Very fast | CI / testing only |
| `qwen3:1.7b` | Good | ~1-2s/request | Default — daily use |
| `qwen3:4b` | Excellent | ~3-5s/request | High-stakes engagements |

---

## Limitations

- **Regex misses contextual data**: bare hostnames (`DC01`), domain usernames (`CONTOSO\user`), cleartext passwords in unusual formats — the LLM layer is essential for these.
- **LLM can miss things in very dense output**: chunks >1500 chars may lose context at boundaries. Tune `LLM_CHUNK_SIZE` if needed.
- **No provable privacy guarantee**: correlation attacks on writing style or query patterns are out of scope. This tool prevents data correlation through content, not metadata.
- **Surrogate collision risk is low but non-zero**: if two different originals happen to get the same surrogate (probabilistically unlikely), deanonymization will be incorrect. The vault prevents this within an engagement.
- **Not a substitute for contract review**: always verify what your NDA/contract allows before using cloud AI on client engagements.
