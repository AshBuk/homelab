# Server Shortcuts

Quick keys & escape hatches for working with the headless server over SSH.

## SSH escape sequences

The SSH client watches for `~` **right after a newline** (press `Enter` first).
Useful when a session freezes because the network blipped (e.g. `apt upgrade`
restarted networkd/WiFi and the link dropped — the server is fine, only the
connection hung).

| Keys              | What it does                                              |
|-------------------|----------------------------------------------------------|
| `Enter` `~` `.`   | **Kill a frozen SSH session** (closes the client locally; server untouched) |
| `Enter` `~` `Ctrl-Z` | Suspend SSH to the background (`fg` to return)         |
| `Enter` `~` `~`   | Send a literal `~` (e.g. when nested through a second ssh) |
| `Enter` `~` `?`   | Show the list of escape sequences                        |

> Order matters: tap **Enter**, then **`~`**, then the action key. If `~.` seems
> to do nothing, you didn't start from a fresh line — hit Enter and retry.

## Recovering a hung connection (no reboot needed!)

1. `Enter` `~` `.` — drop the frozen client, **or** just open a new terminal.
2. `ssh ashbuk@172.20.10.10` — the old hung session doesn't block a new one.
3. The server kept running the whole time. Don't reboot it for a frozen SSH.

## Tips

- Run **tmux on the server**, not locally — then a dropped link doesn't lose your
  work: reconnect → `tmux attach`.
- Add `ServerAliveInterval 30` to `~/.ssh/config` so the client notices drops fast
  instead of hanging.
