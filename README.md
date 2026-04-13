# LLM Anonymization

Transparent anonymization proxy for Claude Code in penetration testing engagements.

Sits between Claude Code and the Anthropic API. Every message, bash output, file read, and grep result is anonymized before leaving your machine. Responses are deanonymized before Claude Code sees them. Claude never touches real client data.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        YOUR MACHINE                             │
│                                                                 │
│  Claude Code                                                    │
│      │  ANTHROPIC_BASE_URL=http://localhost:8080               │
│      ▼                                                          │
│  pentest-proxy (FastAPI :8080)                                  │
│      │                                                          │
│      ├─ Layer 1: LLM Detector (Ollama qwen3:1.7b)              │
│      │   └─ Understands context: hostnames, usernames,         │
│      │      org names, credentials, internal system names      │
│      │                                                          │
│      ├─ Layer 2: Regex Safety Net                               │
│      │   └─ Deterministic: IPs, CIDRs, hashes, MACs,           │
│      │      emails, domains, URLs                               │
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
| Bare hostnames | `DC01`, `FILESERVER-PRD` | LLM |
| Domain usernames | `CONTOSO\jsmith` | LLM |
| Cleartext passwords | `C0nt0s0@2024!` | LLM |
| Organization names | `Contoso Corporation` | LLM |
| Person names | `John Smith` | LLM |
| Internal app/project names | `Project Phoenix` | LLM |
| Sensitive file paths | `/home/jsmith/docs` | LLM |
| Bearer tokens / session IDs | `eyJhbGci...` | LLM |

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

### Option A: Native Python + Native Ollama (recommended for Apple Silicon)

```bash
# 1. Setup
cd pentest-proxy
./scripts/setup.sh

# 2. Pull LLM model (one-time, ~1GB)
ollama pull qwen3:1.7b

# 3. Start proxy (terminal 1)
ENGAGEMENT_ID=client-acme-2026 ./scripts/run.sh

# 4. Use Claude Code (terminal 2)
export ANTHROPIC_BASE_URL=http://localhost:8080
export ENGAGEMENT_ID=client-acme-2026
claude
```

### Option B: Docker (Ollama on Mac, proxy in container)

```bash
# Pull model first (native Ollama must be running)
ollama pull qwen3:1.7b

# Start proxy
ENGAGEMENT_ID=client-acme-2026 docker compose -f docker-compose.native-ollama.yml up -d

# Use Claude Code
export ANTHROPIC_BASE_URL=http://localhost:8080
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
# Unit + regex tests (no Ollama required, runs in seconds)
make test

# Full pipeline tests including LLM (requires Ollama running)
make test-integration
```

The integration test suite enforces a **0% leak policy**: for every pentest fixture (nmap, mimikatz, CrackMapExec, Burp, enum4linux, bash history, LDAP dump, Metasploit), none of the strings in `must_anonymize` may appear in the anonymized output.

---

## Configuration

All settings via environment variables or `.env` file (copy from `.env.example`):

| Variable | Default | Description |
|----------|---------|-------------|
| `ENGAGEMENT_ID` | `default` | Isolates vault per client — **change per engagement** |
| `OLLAMA_HOST` | `http://localhost:11434` | Ollama endpoint |
| `OLLAMA_MODEL` | `qwen3:1.7b` | Use `qwen3:4b` for better quality |
| `LLM_ENABLED` | `true` | Set `false` to run regex-only (faster, less coverage) |
| `OLLAMA_TIMEOUT` | `30` | Seconds before giving up on LLM (falls back to regex) |
| `LLM_CHUNK_SIZE` | `1500` | Chars per chunk for long tool outputs |
| `PORT` | `8080` | Proxy listen port |

---

## LLM Model Selection

| Model | Quality | Speed (CPU) | Use when |
|-------|---------|------------|----------|
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
