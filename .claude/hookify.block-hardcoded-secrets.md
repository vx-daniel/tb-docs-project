---
name: block-hardcoded-secrets
enabled: true
event: file
conditions:
  - field: new_text
    operator: regex_match
    pattern: (API_KEY|SECRET|TOKEN|PASSWORD)\s*[=:]\s*["'][A-Za-z0-9_\-]{16,}
action: block
---
🔐 Hardcoded secret detected. Use environment variables instead.
