# LLM-anonymization
Reverse proxy for Claude Code that anonymizes sensitive pentest data (IPs, hashes, credentials, hostnames,     PII) before it reaches Anthropic. Dual-layer detection: local Ollama LLM + regex safety net, with     per-engagement vault and self-improving feedback loop.
