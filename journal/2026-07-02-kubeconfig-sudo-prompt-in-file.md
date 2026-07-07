# 2026-07-02 sudo password prompt ended up inside kubeconfig

**Symptom.** Pulled the k3s kubeconfig in one go:

```bash
ssh -t <server> 'sudo cat /etc/rancher/k3s/k3s.yaml' > ~/.kube/config
```

The terminal "froze" with a blinking cursor. Typed the password blindly worked, but the file started with `[sudo] password:` (prompted) garbage lines.

**Cause.** The redirect grabs *everything* the remote side prints, sudo's password prompt included.

**Fix.** Strip everything before the YAML:

```bash
sed -i '/^apiVersion: v1/,$!d' ~/.kube/config
```

**Next time:** do it in two steps — `sudo cat` into a home-dir copy on the
server, `scp` it over, `rm` the copy (it holds cluster-admin creds).
