# Internet Pi

A Raspberry Pi configuration for network-wide ad blocking, internet monitoring, dynamic DNS and VPN — all deployed with a single Ansible playbook.

---

## What's included

| Service | URL | Default state |
|---|---|---|
| **Pi-hole** | `http://<pi-ip>/admin` | Enabled |
| **Grafana** | `http://<pi-ip>:3030` | Enabled |
| **Prometheus** | `http://<pi-ip>:9090` | Enabled |
| **Speedtest exporter** | `http://<pi-ip>:9798/metrics` | Enabled |
| **DuckDNS** | — (background updater) | Disabled |
| **WireGuard VPN** (PiVPN) | UDP `<pi-ip>:51820` | Disabled |
| **Shelly Plug monitoring** | — (Grafana dashboard) | Disabled |
| **AirGradient monitoring** | — (Grafana dashboard) | Disabled |
| **Starlink monitoring** | — (Grafana dashboard) | Disabled |

---

## Requirements

- Raspberry Pi 4 or later (32-bit or 64-bit Raspberry Pi OS / Debian Trixie or later)
- Python 3 and pip installed on the Pi
- The playbook is designed to run **locally on the Pi itself**

---

## Quick start

### 1. Install Ansible on the Pi

```bash
sudo apt-get install -y python3-pip
pip3 install ansible
export PATH=$PATH:~/.local/bin
```

### 2. Clone the repository

```bash
git clone https://github.com/Linzell/internet-pi
cd internet-pi
```

### 3. Install Ansible requirements

```bash
ansible-galaxy collection install -r requirements.yml
```

### 4. Create your config files

```bash
cp example.config.yml config.yml
cp example.inventory.ini inventory.ini
```

Edit `inventory.ini` and uncomment the local connection line:

```ini
192.168.1.x ansible_connection=local ansible_user=YOUR_USERNAME
```

Replace `192.168.1.x` with your Pi's IP and `YOUR_USERNAME` with your Linux username.

### 5. Edit config.yml

See the [Configuration reference](#configuration-reference) section below.

### 6. Run the playbook

```bash
ansible-playbook main.yml
```

---

## Configuration reference

Open `config.yml` and set the options for the services you want. All options below are also documented in `example.config.yml`.

### General

```yaml
# Where all service data is stored (~ = your home directory)
config_dir: '~'
```

### Pi-hole

Pi-hole provides network-wide ad blocking and can serve as a DHCP server.

```yaml
pihole_enable: true
pihole_hostname: pihole
pihole_timezone: Europe/Paris        # your timezone
pihole_password: "change-me"         # admin UI password
```

**After deploy:** visit `http://<pi-ip>/admin`

#### DHCP setup (optional)

Pi-hole can hand out IP addresses to all devices on your network.
Enable it in the Pi-hole admin UI: **Settings → DHCP → Enable DHCP server**.

> **Livebox / Orange users:** The Livebox does not allow changing the DNS
> server in its DHCP settings. A reliable workaround is to shrink the
> Livebox DHCP pool to only cover the IPs you need for the router and any
> devices that must use it (e.g. a TV box), then let Pi-hole serve all
> other devices.
>
> Example:
> - Set the Livebox pool to `192.168.1.1–192.168.1.3`
> - `.1` = Livebox, `.2` = TV box, `.3` = Pi (static)
> - Pi-hole pool: `192.168.1.4–192.168.1.100`
>
> Devices outside the Livebox range will automatically fall back to
> Pi-hole's DHCP, which tells them to use `192.168.1.3` as their DNS.

#### Static IP for the Pi

The Pi must always have the same IP. Configure it once:

```bash
sudo nmcli con mod "$(nmcli -f NAME,DEVICE con show --active | awk '/eth0/{print $1}')" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.3/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "127.0.0.1 8.8.8.8"
sudo nmcli con up "$(nmcli -f NAME,DEVICE con show --active | awk '/eth0/{print $1}')"
```

---

### Internet monitoring (Grafana + Prometheus)

Tracks your connection speed, ping latency and uptime over time.

```yaml
monitoring_enable: true
monitoring_grafana_admin_password: "change-me"
monitoring_speedtest_interval: 60m   # how often to run a speed test
monitoring_ping_interval: 5s         # how often to ping hosts

monitoring_ping_hosts:
  - http://www.google.com/;google.com
  - https://github.com/;github.com
  - https://www.apple.com/;apple.com
```

**After deploy:**
- Grafana: `http://<pi-ip>:3030` — login `admin` / your `monitoring_grafana_admin_password`
- Navigate to **Dashboards → Internet connection**
- Trigger a manual speed test: `curl http://localhost:9798/metrics`

> Data builds up over time. Switch the Grafana time range to
> **Last 15 minutes** to see data immediately after first deploy.

---

### DuckDNS (dynamic DNS)

Keeps a free hostname (e.g. `myhome.duckdns.org`) pointed at your home's
public IP — even when your ISP changes it. Required for a stable WireGuard
VPN endpoint.

**Setup:**
1. Sign up free at [duckdns.org](https://www.duckdns.org)
2. Create a subdomain (e.g. `piguard`) → you get `piguard.duckdns.org`
3. Copy your token from the DuckDNS dashboard

```yaml
duckdns_enable: true
duckdns_subdomain: "piguard"                  # without .duckdns.org
duckdns_token: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

The container updates your IP every 5 minutes automatically.

---

### WireGuard VPN (PiVPN)

Gives you encrypted access to your home network from anywhere.

**Prerequisites:**
- DuckDNS configured (recommended) — or a static public IP
- UDP port `51820` forwarded on your router to the Pi's IP

```yaml
wireguard_enable: true
wireguard_host: "auto"       # auto = uses duckdns_subdomain.duckdns.org
                             #        if duckdns_enable is true,
                             #        otherwise detects your public IP
wireguard_port: 51820
wireguard_vpn_net: "10.6.0.0"
wireguard_vpn_subnet: "24"
wireguard_mtu: 1420
wireguard_dns1: "192.168.1.3"   # Pi-hole — ad blocking works over VPN too
wireguard_dns2: "8.8.8.8"
```

> **`wireguard_host: "auto"` resolution order:**
> 1. DuckDNS enabled → uses `duckdns_subdomain.duckdns.org` (permanent ✓)
> 2. DuckDNS disabled → fetches current public IP from api.ipify.org (breaks if IP changes)

**Managing VPN clients after deploy:**

```bash
# Add a new client (shows QR code for mobile apps)
pivpn add

# List all clients
pivpn list

# Show QR code again for an existing client
pivpn qrcode clientname

# Revoke a client
pivpn remove clientname
```

Install the **WireGuard app** on your phone or laptop, scan the QR code, and you're connected.

**Port forwarding on Livebox (Orange):**
Go to **Livebox admin → Network → NAT/PAT** and add a rule:
- Protocol: UDP
- External port: 51820
- Internal IP: `192.168.1.3`
- Internal port: 51820

---

### Prometheus (advanced)

Add custom scrape targets in `config.yml`:

```yaml
prometheus_extra_scrape_configs: |
  - job_name: 'my-server'
    scrape_interval: 15s
    static_configs:
      - targets: ['192.168.1.10:9100']

prometheus_node_exporter_targets:
  - 'nodeexp:9100'
  - 'another-pi.local:9100'   # add more Pis here
```

---

### Optional services

#### Shelly Plug monitoring

```yaml
shelly_plug_enable: true
shelly_plug_hostname: 192.168.1.50
shelly_plug_http_username: admin
shelly_plug_http_password: "change-me"
```

#### AirGradient air quality monitoring

```yaml
airgradient_enable: true
airgradient_sensors:
  - id: livingroom
    ip: "192.168.1.60"
    port: 9925
```

#### Starlink monitoring

```yaml
starlink_enable: true
```

---

## Updating services

Re-run the playbook to apply any config changes:

```bash
ansible-playbook main.yml
```

To pull the latest Docker images for a service:

```bash
cd ~/internet-monitoring   # or ~/pi-hole, ~/duckdns, etc.
docker compose pull
docker compose up -d --no-deps
docker system prune -af    # clean up old images
```

---

## Uninstall

```bash
cd ~/internet-monitoring && docker compose down -v
cd ~/pi-hole              && docker compose down -v
cd ~/duckdns              && docker compose down -v

# Remove any other optional services the same way:
# cd ~/shelly-plug-prometheus && docker compose down -v
# cd ~/starlink-exporter      && docker compose down -v

docker system prune -af
```

Then delete the service directories and you're clean.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `permission denied` on Docker socket | Log out and back in (group membership refresh), then re-run |
| Grafana shows all zeros | Change time range to **Last 15 min**; data needs time to accumulate |
| Pi-hole DHCP shows no leases | Make sure router DHCP pool is exhausted or disabled |
| Speedtest shows `speedtest_up 0` | DNS issue — the service has `dns: [8.8.8.8]` to fix this; restart the container |
| WireGuard clients can't connect | Check port 51820/UDP is forwarded on the router; check `pivpn list` |
| DuckDNS not updating | Verify token and subdomain in `config.yml`; check `docker logs duckdns` |

---

## License

MIT

## Author

Originally created in 2021 by [Jeff Geerling](https://www.jeffgeerling.com/).
Extended with Trixie support, DuckDNS, WireGuard/PiVPN and DHCP improvements.
