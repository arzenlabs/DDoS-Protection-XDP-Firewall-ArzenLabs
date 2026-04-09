# XDP Port Range Deploy (25566-25665)

Deployment-ready GitHub repository for running an **XDP loader** on Linux servers that host game or proxy traffic on the port range **25566-25665**.

This repo is meant for providers, infrastructure engineers, and game hosting operators who want a clean way to deploy the supplied `xdp-loader` binary, expose metrics, and standardize server-side setup for services such as:

- Minecraft Java
- Minecraft Bedrock
- Proxy/API/query/status endpoints
- Multi-tenant game hosting on a shared protected IP

## What this repo does

- ships the provided `xdp-loader` binary
- installs it under `/opt/xdp-port-range-deploy`
- runs it through `systemd`
- exposes a metrics endpoint
- documents how to prepare the server for services listening on **25566-25665**

## Important note

The included `xdp-loader` binary accepts an interface and optional metrics bind address:

```bash
xdp-loader [OPTIONS] [INTERFACE]
```

That means this repository is a **deployment and operations template**. The actual packet policy or per-port decision logic depends on the XDP program / backend logic used with your environment.

So this repo solves:

- install
- service management
- metrics
- standard port-range deployment guidance

It does **not** claim to embed all custom filtering logic for `25566-25665` by itself unless your backend XDP implementation already supports that.

---

## Repository layout

```text
xdp-port-range-deploy/
├── README.md
├── LICENSE
├── .gitignore
├── config.env.example
├── bin/
│   └── xdp-loader
├── docs/
│   ├── architecture.md
│   └── setup.md
├── scripts/
│   ├── install.sh
│   ├── start.sh
│   ├── stop.sh
│   └── verify.sh
└── systemd/
    └── xdp-loader.service
```

---

## Recommended port plan

| Port(s) | Suggested use |
|---|---|
| 25566 | Minecraft Java main port |
| 25567 | Minecraft Bedrock main port |
| 25568 | Query / API / status |
| 25569-25665 | Additional clients, proxies, or backend allocations |

Keeping Java, Bedrock, and API/query separated is strongly recommended.

---

## Quick deploy

### 1) Clone the repo

```bash
git clone https://github.com/arzenlabs/xdp-port-range-deploy.git
cd xdp-port-range-deploy
```

### 2) Copy config

```bash
cp config.env.example config.env
nano config.env
```

Example:

```env
INTERFACE=eth0
PORT_START=25566
PORT_END=25665
METRICS_ADDR=0.0.0.0:9090
INSTALL_DIR=/opt/xdp-port-range-deploy
```

### 3) Install

```bash
chmod +x scripts/*.sh
sudo ./scripts/install.sh
```

### 4) Start service

```bash
sudo systemctl enable xdp-loader
sudo systemctl start xdp-loader
```

### 5) Verify

```bash
sudo systemctl status xdp-loader
sudo ./scripts/verify.sh
```

---

## Manual operation

### Start manually

```bash
sudo ./scripts/start.sh
```

### Stop manually

```bash
sudo ./scripts/stop.sh
```

---

## Metrics

If `METRICS_ADDR` is configured, the service starts with:

```bash
xdp-loader --metrics-addr 0.0.0.0:9090 eth0
```

You can then test locally:

```bash
curl http://127.0.0.1:9090/
```

If you expose metrics publicly, protect it behind a firewall or reverse proxy ACL.

---

## Server preparation for 25566-25665

Your backend services still need to listen on the intended ports.

Useful checks:

```bash
ss -tulnp | grep 25566
ss -tulnp | grep 25665
ss -tulnp | egrep '2556[6-9]|256[0-5][0-9]|2566[0-5]'
```

To inspect traffic:

```bash
sudo tcpdump -ni any 'portrange 25566-25665'
```

---

## Firewall example

This repo does not automatically alter your firewall, but you can allow the range explicitly.

### UFW

```bash
sudo ufw allow 25566:25665/tcp
sudo ufw allow 25566:25665/udp
```

### iptables

```bash
sudo iptables -A INPUT -p tcp --match multiport --dports 25566:25665 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 25566:25665 -j ACCEPT
```

### nftables

```nft
add rule inet filter input tcp dport 25566-25665 accept
add rule inet filter input udp dport 25566-25665 accept
```

---

## Common issues

### Players cannot connect

Check all of these:

- the backend process is actually listening on the target port
- your firewall allows TCP and/or UDP as needed
- your provider anti-DDoS profile is not silently dropping the traffic
- Bedrock and Java are not accidentally sharing incompatible handling
- status/query traffic is not mixed incorrectly with gameplay traffic

### XDP service starts but traffic still fails

That usually means the loader is running, but one of the following is wrong:

- interface selected is incorrect
- upstream routing or anti-DDoS rules are wrong
- backend service bind address is wrong
- NAT / DNAT rules conflict
- the underlying XDP program is not implementing the port logic you expect

### Metrics do not open

Check:

```bash
sudo journalctl -u xdp-loader -n 100 --no-pager
ss -ltnp | grep 9090
```

---

## Publishing to GitHub

```bash
git init
git add .
git commit -m "Initial XDP port-range deployment repo"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/xdp-port-range-deploy.git
git push -u origin main
```

---

## License

MIT
