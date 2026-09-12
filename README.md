# Self-Hosted WireGuard VPN on Vultr (Ubuntu 24.04)

A personal VPN server built from scratch on a Vultr VPS running Ubuntu 24.04 LTS, using WireGuard for the tunnel. Supports multiple client devices (PC, phone, laptop) connecting simultaneously.

## Stack

- **VPS Provider:** Vultr
- **OS:** Ubuntu 24.04 LTS
- **VPN protocol:** WireGuard (plain CLI, no Docker/wg-easy)
- **Firewall:** UFW (OS-level) + Vultr Cloud Firewall (network-level)

## Server Setup

### 1. Provision the VPS
- Deploy Ubuntu 24.04 LTS on Vultr
- Add an SSH key at deploy time (no password login)
- Enable "Limited User Login" for a non-root sudo user
- Skip paid add-ons (Auto Backups, DDoS Protection) — not needed for personal use

### 2. Harden SSH
- Log in as the sudo user, not root
- Disable root SSH login and password auth in `/etc/ssh/sshd_config`

### 3. Install WireGuard

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wireguard qrencode -y
```

### 4. Generate the server keypair

```bash
sudo -i
cd /etc/wireguard
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
```

### 5. Enable IP forwarding

In `/etc/sysctl.conf`, uncomment:
```
net.ipv4.ip_forward=1
```
Apply with `sysctl -p`.

### 6. Find the public network interface

```bash
ip route | grep default
```
Note the interface name (e.g. `eth0`, `enp1s0`) — used in the NAT rule below.

### 7. Create `/etc/wireguard/wg0.conf`

```ini
[Interface]
PrivateKey = <SERVER_PRIVATE_KEY>
Address = 10.8.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -A FORWARD -o wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o <INTERFACE> -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -D FORWARD -o wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o <INTERFACE> -j MASQUERADE
```

### 8. Open the firewall and start the interface

```bash
ufw allow 51820/udp
ufw allow OpenSSH
ufw enable

wg-quick up wg0
systemctl enable wg-quick@wg0
```

Also confirm the Vultr Cloud Firewall (if enabled) allows inbound UDP 51820 and TCP 22.

## Adding a Client Device

Repeat per device, incrementing the internal IP each time (`10.8.0.2`, `10.8.0.3`, `10.8.0.4`, ...).

### 1. Generate a keypair for the device

```bash
wg genkey | tee device_private.key | wg pubkey > device_public.key
```

### 2. Add it as a peer on the server

Append to `/etc/wireguard/wg0.conf`:

```ini
[Peer]
PublicKey = <DEVICE_PUBLIC_KEY>
AllowedIPs = 10.8.0.X/32
```

Apply live without a restart:

```bash
wg syncconf wg0 <(wg-quick strip wg0)
```

### 3. Build the client config

```ini
[Interface]
PrivateKey = <DEVICE_PRIVATE_KEY>
Address = 10.8.0.X/32
DNS = 1.1.1.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <VPS_PUBLIC_IP>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

> Double-check there's no stray whitespace in the `Endpoint` line — a space before the port causes a "No such host is known" error in the WireGuard client.

### 4. Import into the client

- **Desktop (Windows/macOS/Linux):** Install the official WireGuard app, import the `.conf` file directly, activate the tunnel.
- **Mobile (iOS/Android):** Generate a QR code on the server and scan it with the WireGuard app:
  ```bash
  qrencode -t ansiutf8 < device_client.conf
  ```

### 5. Verify

Visit `https://whatismyipaddress.com` on the connected device — it should show the VPS's public IP, not the device's real IP.

## Revoking a Device

Delete its `[Peer]` block from `/etc/wireguard/wg0.conf`, then re-apply:

```bash
wg syncconf wg0 <(wg-quick strip wg0)
```

## Known Issues

- **Datacenter IP flags:** Google/YouTube and similar services sometimes treat traffic from cloud/VPS IP ranges with more suspicion than residential ISPs, occasionally leading to CAPTCHAs or playback issues. This is a known tradeoff of self-hosting vs. commercial VPN providers.
- **App vs. browser behavior on mobile:** Native apps (e.g. YouTube app) can behave differently over the tunnel than the same site loaded in a mobile browser, particularly on WiFi where MTU/path differences from cellular don't apply. If an app misbehaves specifically on WiFi while the browser is fine, it's worth comparing MTU settings or testing the same app over cellular data with the VPN active to isolate whether it's app-specific network handling.

## Security Notes

- Never commit `*.key`, `*.conf` files, or real IP addresses to a public repo — this README uses placeholders (`<SERVER_PUBLIC_KEY>`, `<VPS_PUBLIC_IP>`, etc.) intentionally.
- Rotate any key that has ever been pasted into a chat, log, or screenshot.
- Add a `.gitignore` entry for `*.key` and `*.conf` if you keep your actual configs in the same repo as this documentation.
