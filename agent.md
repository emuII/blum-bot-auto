# Setup Hermes + 9Router + NVIDIA NIM (dari nol)

Tutorial untuk install dan integrasi 3 komponen di VPS Linux (Ubuntu 22/24 atau Debian).

---

## 0. Prasyarat

- VPS Linux (Ubuntu 22.04/24.04 paling enak), root atau sudo
- RAM minimal 2GB (4GB recommended)
- Port 20128 terbuka (firewall + cloud security group)
- Domain optional (untuk dashboard permanen)
- Akun Cloudflare (free) — optional, untuk tunnel permanen

```bash
sudo apt update && sudo apt install -y curl git build-essential ufw
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 20128/tcp
ufw enable
```

---

## 1. Install Node.js (versi stabil)

Pakai `n` atau `nvm`. Recommended: Node 20 LTS atau 22.

```bash
# Cara cepat — install Node 22
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
node -v   # harus v22.x
npm -v
```

---

## 2. Install 9Router

9Router = AI proxy router. Gabungin banyak provider (OpenAI, Anthropic, NVIDIA, dll) jadi 1 endpoint dengan smart fallback.

```bash
sudo npm install -g 9router
9router --version
# 9router 0.4.13 (terbaru per Mei 2026)

# Run sekali manual buat init config & DB
9router
# Listen: http://localhost:20128
# Akses dashboard pertama kali via http://<vps-ip>:20128
# Set password admin saat first login
```

Stop sementara dengan Ctrl+C.

### Setup systemd service supaya auto-start

```bash
sudo tee /etc/systemd/system/9router.service > /dev/null <<'EOF'
[Unit]
Description=9Router AI Router
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/9router
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now 9router
sudo systemctl status 9router
```

### Akses dashboard

- **Lokal**: http://localhost:20128
- **Public via cloudflared quick tunnel** (gratis, URL random):

```bash
# Install cloudflared
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared

# Buat systemd service untuk tunnel
sudo tee /etc/systemd/system/9router-tunnel.service > /dev/null <<'EOF'
[Unit]
Description=9Router Cloudflare Tunnel
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/cloudflared tunnel --url http://localhost:20128 --no-autoupdate
Restart=always
RestartSec=10
StandardOutput=append:/tmp/9router-tunnel.log
StandardError=append:/tmp/9router-tunnel.log

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable --now 9router-tunnel
sleep 5
tail -50 /tmp/9router-tunnel.log | grep trycloudflare.com
# Output: https://xxxxx-yyyyy.trycloudflare.com
```

URL random itu yang dibuka di browser PC.

⚠️ **Catatan**: Quick tunnel URL berubah tiap restart. Kalau mau permanen, pakai Cloudflare named tunnel:

```bash
cloudflared tunnel login           # login Cloudflare
cloudflared tunnel create 9router-prod
cloudflared tunnel route dns 9router-prod 9router.yourdomain.com
# Edit ~/.cloudflared/config.yml dengan ingress rule → http://localhost:20128
cloudflared tunnel run 9router-prod
```

---

## 3. Add NVIDIA NIM ke 9Router

NVIDIA NIM = serverless inference gratis untuk dev. OpenAI-compatible. Buat akun di https://build.nvidia.com → klik avatar → "Get API Key" → simpan key (`nvapi-...`).

### Cara 1: lewat Dashboard (gampang)

1. Buka 9router dashboard di browser
2. Login pake password yg lo set
3. Klik **"+ Add OpenAI Compatible"**
4. Isi:
   - **Name**: NVIDIA NIM
   - **Prefix**: `nvidia`
   - **Base URL**: `https://integrate.api.nvidia.com/v1`
   - **API Key**: `nvapi-...` (paste yg dari NVIDIA)
   - **API Type**: `chat`
5. Klik **Check** — harus return `valid: true`
6. Klik **Save**

### Cara 2: lewat API (script-friendly)

```bash
# Get CLI token (deterministic dari machine ID VPS)
TOKEN=$(node -e '
const crypto = require("crypto");
const {machineIdSync} = require("node-machine-id");
const t = crypto.createHash("sha256")
  .update(machineIdSync() + "9r-cli-auth")
  .digest("hex").substring(0, 16);
console.log(t);
')
echo "Token: $TOKEN"

# Validate first (no side effects)
curl -s -H "x-9r-cli-token: $TOKEN" -H "Content-Type: application/json" \
  -X POST http://localhost:20128/api/provider-nodes/validate \
  -d '{"baseUrl":"https://integrate.api.nvidia.com/v1","apiKey":"nvapi-XXX","type":"openai-compatible","modelId":"meta/llama-3.3-70b-instruct"}'
# Expected: {"valid":true}

# Create node
NODE_ID=$(curl -s -H "x-9r-cli-token: $TOKEN" -H "Content-Type: application/json" \
  -X POST http://localhost:20128/api/provider-nodes \
  -d '{"type":"openai-compatible","apiType":"chat","name":"NVIDIA NIM","prefix":"nvidia","baseUrl":"https://integrate.api.nvidia.com/v1"}' \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['node']['id'])")
echo "Node: $NODE_ID"

# Create connection (attach API key)
curl -s -H "x-9r-cli-token: $TOKEN" -H "Content-Type: application/json" \
  -X POST http://localhost:20128/api/providers \
  -d "{\"provider\":\"$NODE_ID\",\"apiKey\":\"nvapi-XXX\",\"name\":\"NVIDIA NIM\"}"
```

### Test setelah add

Get API key dari dashboard → Settings → API Keys → bikin new key (atau pakai existing).

```bash
NINER_KEY="sk-eec065249f6b85df-aqzhoh-695a72e4"   # dari 9router

curl -X POST http://43.156.31.182:20128/v1/chat/completions \
  -H "Authorization: Bearer sk-eec065249f6b85df-aqzhoh-695a72e4" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/meta/llama-3.3-70b-instruct",
    "messages": [{"role":"user","content":"halo"}],
    "max_tokens": 50
  }'
```

Harus return chat completion JSON.

### Add NVIDIA models ke smart fallback combo

Di dashboard:
1. Tab **Combos** → pilih combo lo (misal `free_smart_fallback`)
2. **Add Model** → pilih dari list `nvidia/...`
3. Drag posisi yang lo mau

Top NVIDIA models (recommended urutan kuat → ringan):
- `nvidia/deepseek-ai/deepseek-v4-pro`
- `nvidia/deepseek-ai/deepseek-v3.1-terminus`
- `nvidia/moonshotai/kimi-k2-thinking`
- `nvidia/meta/llama-3.3-70b-instruct`
- `nvidia/google/gemma-4-31b-it`

---

## 4. Install Hermes (Telegram bot frontend)

Hermes = Telegram bot yang punya akses ke shell, file edit, browser dll. Pakai 9router sebagai LLM backend.

```bash
# Pre-req: pip
sudo apt install -y python3-pip python3-venv

# Hermes punya bin/cli (assuming hermes-agent package)
# Cek docs internal lo, tapi general:
mkdir -p /root/.hermes && cd /root/.hermes
npm install hermes-agent   # atau: pip install hermes-agent

# Set env vars di /root/.hermes/.env
cat > /root/.hermes/.env <<'EOF'
TELEGRAM_BOT_TOKEN=<from @BotFather>
TELEGRAM_OWNER_ID=<your telegram user id>
OPENAI_API_KEY=<9router api key dari step 3>
OPENAI_BASE_URL=http://localhost:20128/v1
DEFAULT_MODEL=free_smart_fallback
EOF
chmod 600 /root/.hermes/.env
```

Edit config:

```bash
cat > /root/.hermes/config.yaml <<'EOF'
providers:
  9router:
    name: 9Router
    base_url: http://localhost:20128/v1
    key_env: OPENAI_API_KEY
    default_model: free_smart_fallback
    models:
      free_smart_fallback: {}
      nvidia/deepseek-ai/deepseek-v4-pro: {}
      nvidia/meta/llama-3.3-70b-instruct: {}
fallback_providers:
  - provider: 9router
    model: free_smart_fallback
EOF
chmod 600 /root/.hermes/config.yaml
```

Service:

```bash
sudo tee /etc/systemd/system/hermes.service > /dev/null <<'EOF'
[Unit]
Description=Hermes Agent (Telegram bot)
After=network.target 9router.service

[Service]
Type=simple
User=root
WorkingDirectory=/root/.hermes
EnvironmentFile=/root/.hermes/.env
ExecStart=/usr/bin/node /root/.hermes/node_modules/.bin/hermes-agent
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now hermes
journalctl -u hermes -f
```

Test: kirim `/start` ke bot lo di Telegram. Harus respond.

---

## 5. Verify pipeline end-to-end

Di Telegram:
```
/model
```
Pilih `9Router` → `free_smart_fallback`. Lalu kirim:
```
test 1+1
```
Bot harus respond. Kalau pake `/model` → `nvidia/meta/llama-3.3-70b-instruct`, request akan langsung ke NVIDIA endpoint.

---

## 6. Maintenance & monitoring

```bash
# Cek service
systemctl status 9router hermes 9router-tunnel

# Cek logs
journalctl -u 9router -f
journalctl -u hermes -f
tail -f /tmp/9router-tunnel.log

# Update 9router (kalau ada versi baru)
sudo npm install -g 9router@latest
sudo systemctl restart 9router

# Backup config
tar czf hermes-9router-backup-$(date +%Y%m%d).tar.gz \
  /root/.hermes/config.yaml /root/.hermes/.env \
  /root/.9router/db.json
```

---

## 7. Troubleshooting

| Masalah | Fix |
|---|---|
| Dashboard ga keload | Cek `systemctl status 9router`. Cek port 20128 listening. Cek firewall+cloud security group. |
| "API key unauthorized" saat Check NVIDIA | Test dulu pake curl direct ke `https://integrate.api.nvidia.com/v1/models`. Kalau OK, restart 9router. |
| Hermes ga respond di Telegram | Cek `OPENAI_BASE_URL=http://localhost:20128/v1` di .env. Cek API key 9router masih aktif. |
| Tunnel URL hilang | `systemctl restart 9router-tunnel && tail /tmp/9router-tunnel.log`. URL baru muncul di log. |
| 429 rate limit | Smart fallback otomatis switch ke model berikutnya. Kalau semua kena, tunggu atau add provider lain. |

---

## Catatan biaya

- **9router**: GRATIS (open source, self-hosted)
- **NVIDIA NIM**: GRATIS untuk dev (rate-limited tapi cukup untuk personal)
- **VPS**: tergantung provider — Tencent Lighthouse SGP $5/bulan, AWS t3.small ~$15
- **Cloudflare tunnel**: GRATIS (named atau quick)
- **Telegram bot**: GRATIS

Total bulanan: ~$5-15 hanya VPS.

---

## Resources

- 9Router docs: https://github.com/9router/9router (atau npm page)
- NVIDIA NIM: https://build.nvidia.com/explore/discover
- NVIDIA API ref: https://docs.api.nvidia.com/nim/reference/llm-apis
- Cloudflared: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/
