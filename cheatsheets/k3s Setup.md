# k3s Setup (Ubuntu Server)

Single-node k3s install + remote kubectl from the workstation.

## 1. Firewall (UFW) — before installing k3s

Crucial for communication between Pods and the API server.

```bash
sudo ufw allow 6443/tcp                  # Kubernetes API (kubectl access)
sudo ufw allow from 10.42.0.0/16 to any  # Pod network (Flannel subnet)
sudo ufw allow from 10.43.0.0/16 to any  # Service network
```

## 2. Install k3s

```bash
curl -sfL https://get.k3s.io | sh -

# Verify (STATUS must be Ready)
sudo k3s kubectl get nodes
```

The script installs a single binary to `/usr/local/bin/k3s` (`kubectl`,
`crictl`, `ctr` are symlinks to it), creates the `k3s.service` systemd unit
(enabled → survives reboot) and a full uninstall script `k3s-uninstall.sh`.

## 3. kubectl on the workstation

1. Install the client (Fedora): `sudo dnf install kubernetes-client`
2. Copy kubeconfig — the file is root-only on the server, plain `scp` gets
   `permission denied`. Two-step way that works cleanly:

   ```bash
   # on the server:
   sudo cat /etc/rancher/k3s/k3s.yaml > ~/k3s.yaml

   # from the workstation:
   scp <user>@<server-ip>:k3s.yaml ~/.kube/config

   # on the server (the file holds cluster-admin creds):
   rm ~/k3s.yaml
   ```

   > One-liner `ssh -t ... 'sudo cat ...' > file` writes the sudo prompt
   > *into* the file — see
   > `journal/2026-07-02-kubeconfig-sudo-prompt-in-file.md`.

3. Edit `~/.kube/config`:
   - replace `127.0.0.1` with `<server-ip>`
   - `chmod 600 ~/.kube/config` (contains the cluster-admin certificate)

## 4. Validate & first namespace

```bash
kubectl get nodes               # from the workstation — full chain check
kubectl get pods -A             # core components (see k3s Default Components.md)
kubectl create namespace homelab
```

## Networking & IP changes

- k3s binds to the server IP at installation — it goes into the TLS
  certificate and the kubeconfig.
- If the server IP changes (e.g. new network / DHCP), the kubeconfig must be
  updated and the certificate may become invalid.
- Anticipating an IP change? Install with the `--tls-san <name-or-ip>` flag.
