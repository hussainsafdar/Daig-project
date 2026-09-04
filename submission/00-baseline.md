# Phase 0 — Baseline

## Task 1 & 2: `make hooks` + `make check`

Note: on a fresh macOS machine, `make check` first failed with
`ModuleNotFoundError: No module named 'yaml'` because PyYAML is not installed by
default with system Python 3. Fixed with:
`pip3 install pyyaml --break-system-packages`. This is a real environment
dependency gap not called out in the README's quick-start.

## Task 3: Verified vs Unverified

`VERIFICATION.md` draws a hard line between three claims: **VERIFIED (real)**
(the code was actually executed and its output inspected), **VERIFIED
(static)** (a tool parsed or syntax-checked it without running it), and
**UNVERIFIED** (only read by eye, never run at all). This distinction matters
because a clean-looking repository can hide the fact that entire subsystems —
Terraform, OpenBao's AppRole flow, Semgrep's custom rules, the Swarm stack —
were never actually exercised before being handed to interns. "It should work"
and "I ran it" are different claims with very different failure risk, and
conflating them is exactly how a misconfiguration survives until the worst
possible moment (in this repo's own framing: the forty minutes before iftar).
Treating the two as interchangeable would have cost real time in this
rotation: `VERIFICATION.md` explicitly flagged the OpenBao AppRole flow and
Compose `--env-file` precedence as the most likely things to break — and both
did, exactly as predicted, during the Phase 6 (secrets) work done ahead of
this baseline task.
