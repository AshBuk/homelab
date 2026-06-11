# 2026-06-10 — tmux won't start: `missing or unsuitable terminal: xterm-kitty`

**Symptom.** After installing tmux on the server, `tmux` failed with
`missing or unsuitable terminal: xterm-kitty`.

**Cause.** The workstation runs the **Kitty** terminal, which sets
`TERM=xterm-kitty`. SSH forwards that value to the server, but the server has no
terminfo entry for `xterm-kitty`, so tmux refuses to start.

**Fix (homelab approach — copy the terminfo from the workstation to the server).**
Run once per new server, from the workstation:
```bash
infocmp -x xterm-kitty | ssh user@ip tic -x -
```
After this, `tmux` works as-is.

**Quick fallback** (no install needed): `TERM=xterm-256color tmux`.

