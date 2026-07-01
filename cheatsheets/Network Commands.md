# Network Commands

Operational commands for the server's network link. Reference/status lives in
`Server Commands.md`; the "how it works" lesson in `Network.md`.

## Diagnostics — "what's the state?"

```bash
ip a                                     # interfaces & IPs
ip link                                  # list interfaces / find WiFi name
ip route | grep default                  # default gateway / route
networkctl status wlp3s0                 # link state, AP, online status
sudo iw dev wlp3s0 link                  # what AP we're joined to right now
sudo iw dev wlp3s0 scan | grep -i ssid   # what networks the card can see
ss -tlnp | grep ':22'                    # confirm SSH is listening on :22
```

## Bring WiFi up (netplan)

```bash
sudo nano /etc/netplan/50-wifi.yaml          # SSID + password live here
sudo netplan apply
sudo netplan apply --debug 2>&1 | tail -20   # if it fails
systemctl status netplan-wpa-wlp3s0.service  # WiFi association service
```

## Restart the network stack (when `netplan apply` isn't enough)

`netplan apply` only regenerates config — it doesn't always bounce the link or
daemons. Escalate from soft to hard:

```bash
# 1. Restart the networking daemons (the "harder than netplan" step)
sudo systemctl restart systemd-networkd
sudo systemctl restart netplan-wpa-wlp3s0.service   # WiFi association daemon
sudo systemctl restart systemd-resolved            # DNS resolves but no internet

# 2. Bounce the interface itself (down/up)
sudo ip link set wlp3s0 down
sudo ip link set wlp3s0 up
sudo netplan apply

# 3. Reload the WiFi driver — hardest reset short of a reboot
sudo modprobe -r ath9k && sudo modprobe ath9k
sudo netplan apply

# 4. Last resort
sudo reboot
```

> Before restarting anything, check it isn't the **hotspot**: `networkctl status
> wlp3s0` showing `no-carrier` / `AP (null)` usually means the iPhone hotspot
> slept or dropped to 5 GHz. Restarting the stack won't help — wake the phone
> (open the Personal Hotspot screen, keep "Maximize Compatibility" on).

## Connectivity check (from the workstation)

```bash
ssh ashbuk@172.20.10.10          # connect to the server
ping 172.20.10.10                # reachable? (must be on the same network)
```

## See also

- `Server Commands.md` — current values, interfaces, SSH/service checks.
- `Network.md` — how to read `ip a` / `networkctl status`.
- `journal/2026-06-11-wifi-2.4ghz-vs-iphone-hotspot.md` — the 2.4 vs 5 GHz incident.
