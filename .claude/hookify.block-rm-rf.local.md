---
name: block-dangerous-rm
enabled: true
event: bash
pattern: rm\s+-rf\s+.*(/|~)
action: block
---
🛑 rm -rf with root or home path detected. Blocked.
