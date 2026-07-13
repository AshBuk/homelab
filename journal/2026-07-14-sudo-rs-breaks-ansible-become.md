# 2026-07-14 sudo-rs breaks Ansible privilege escalation

First run of the new Ansible playbook died on Gathering Facts: "Timeout
waiting for privilege escalation prompt". Network fast, sudo fast — but
the prompt looked odd: `[sudo: authenticate] Password:`. That's sudo-rs,
the Rust rewrite, Ubuntu's default since 25.10.

Ansible passes its own marker via `sudo -p` and waits for a line that
*starts with* it. sudo-rs accepts `-p` but wraps it in its own frame —
`[sudo: <marker>] Password:` — so the match never fires and the password
is never sent. Same flags, slightly different output, automation dead.

Tried `apt install sudo` to swap it back — already installed. Both
coexist: `/usr/bin/sudo` is an update-alternatives symlink, sudo-rs wins
50 vs 40, classic lives at `/usr/bin/sudo.ws`. Left the host alone and
pinned `ansible_become_exe: /usr/bin/sudo.ws` in the inventory instead —
the workaround is version-controlled, the distro keeps its default.

Lesson: when automation hangs "waiting for a prompt", diff the prompt it
expects against the one actually printed. Rust rewrites keep the flags,
not the exact output.
