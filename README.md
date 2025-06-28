# 🧠 XMRig Auto Mining + Telegram Bot + Suhu Control (Ubuntu Server)

Tutorial lengkap untuk menjalankan rig mining XMRig secara otomatis dengan fitur:

✅ Auto start mining saat suhu < batas
✅ Stop mining saat suhu tinggi
✅ Kirim notifikasi ke Telegram
✅ Bisa dikontrol via Telegram (`cek RIG`)
✅ Tanpa perlu kontrol pusat
✅ Sudah support `msr`, `huge pages`, dan custom `xmrig.sh`

---

## 📦 1. Update & Install Paket Penting

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git build-essential cmake hwloc libhwloc-dev libuv1-dev libssl-dev curl bc jq lm-sensors -y
```

---

## 🧪 2. Aktifkan Sensor CPU

```bash
sudo sensors-detect --auto
sudo systemctl restart kmod
```

---

## 🔧 3. Clone & Compile XMRig

```bash
cd ~
git clone https://github.com/xmrig/xmrig.git
cd xmrig
mkdir build && cd build
cmake ..
make -j$(nproc)
```

---

## ⚙️ 4. Buat File `xmrig.sh` (Manual Launch Script)

Simpan di: `/home/goppe/xmrig/build/xmrig.sh`

```bash
#!/bin/bash
sudo modprobe msr

./xmrig \
  -o pool.supportxmr.com:3333 \
  -u 42L5wvMMffV29yRRR6ojzX363dWaqtRS1Rpx9hP9PdnidhRtXkikZdvRJuqx4kov3n7YX6ZFa1yA2JhGGinP745g6MmsUEC \
  -p PC-Mint \
  --donate-level 1 \
  --algo rx/0 \
  --threads=6 \
  --cpu-affinity=0,1,2,3,6,7 \
  --no-color \
  --huge-pages \
  --randomx-1gb-pages \
  --asm=auto
```

Lalu jadikan executable:

```bash
chmod +x xmrig.sh
```

---

## 🤖 5. Siapkan Telegram Bot

1. Buka [@BotFather](https://t.me/BotFather), buat bot baru → dapatkan **token**
2. Kirim pesan apa saja ke bot kamu → lalu ambil chat ID dengan:

```bash
curl -s "https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates"
```

---

## 📜 6. Buat `automation.sh` untuk Kontrol Otomatis

Letakkan di `/home/goppe/xmrig/build/automation.sh`

Isi contoh:

```bash
#!/bin/bash

TOKEN="ISI_TOKEN"
CHAT_ID="ISI_CHAT_ID"
RIG_NAME="[G-i7-4770]"

STATUS_FILE="/tmp/xmrig_running"
LAST_CMD_TIME_FILE="/tmp/last_cmd_time"
PREV_STATUS="unknown"

send_message() {
  curl -s -X POST "https://api.telegram.org/bot$TOKEN/sendMessage" \
       -d chat_id="$CHAT_ID" \
       -d text="$RIG_NAME $1"
}

check_temp() {
  sensors | awk '/Package id 0/ {gsub(/[^0-9.]/,"",$4); print $4}'
}

check_xmrig_status() {
  systemctl is-active xmrig >/dev/null 2>&1 && echo "active" || echo "inactive"
}

start_xmrig() { systemctl start xmrig; }
stop_xmrig() { systemctl stop xmrig; }

check_telegram_command() {
  UPDATES=$(curl -s "https://api.telegram.org/bot$TOKEN/getUpdates")
  MESSAGE=$(echo "$UPDATES" | jq -r '.result[-1].message.text')
  DATE=$(echo "$UPDATES" | jq -r '.result[-1].message.date')

  if [[ "$MESSAGE" == "cek ${RIG_NAME//[\[\]]/}" ]]; then
    LAST=$(cat "$LAST_CMD_TIME_FILE" 2>/dev/null || echo 0)
    if (( DATE > LAST )); then
      TEMP=$(check_temp)
      STATUS=$(check_xmrig_status)
      send_message "✅ STATUS: $STATUS\n🌡️ SUHU: ${TEMP}°C"
      echo "$DATE" > "$LAST_CMD_TIME_FILE"
    fi
  fi
}

send_message "💡 Rig hidup. Automation aktif."

while true; do
  TEMP=$(check_temp)
  STATUS=$(check_xmrig_status)

  if (( $(echo "$TEMP > 65" | bc -l) )) && [[ "$STATUS" == "active" ]]; then
    send_message "❌ SUHU $TEMP°C terlalu tinggi. Stop mining."
    stop_xmrig
  elif (( $(echo "$TEMP < 50" | bc -l) )) && [[ "$STATUS" == "inactive" ]]; then
    send_message "✅ SUHU $TEMP°C aman. Start mining."
    start_xmrig
  fi

  if [[ "$STATUS" != "$PREV_STATUS" ]]; then
    send_message "ℹ️ XMRig status: $STATUS"
    echo "$STATUS" > "$STATUS_FILE"
    PREV_STATUS="$STATUS"
  fi

  check_telegram_command &
  sleep 10
done
```

Lalu:

```bash
chmod +x automation.sh
```

---

## 🪪 7. Buat `xmrig.service`

File: `/etc/systemd/system/xmrig.service`

```ini
[Unit]
Description=XMRig Miner Service
After=network.target

[Service]
ExecStart=/home/goppe/xmrig/build/xmrig.sh
WorkingDirectory=/home/goppe/xmrig/build
Restart=always
User=root
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
```

---

## ⚙️ 8. Buat `automation.service`

File: `/etc/systemd/system/automation.service`

```ini
[Unit]
Description=Mining Automation Service
After=network.target

[Service]
ExecStart=/bin/bash /home/goppe/xmrig/build/automation.sh
Restart=always
User=root
Nice=10

[Install]
WantedBy=multi-user.target
```

---

## ▶️ 9. Enable Systemd Service

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable xmrig
sudo systemctl enable automation
sudo systemctl start automation
```

---

## 🧪 10. Tes Bot Telegram

Ketik:

```
cek G-i7-4770
```

Balasan:

```
✅ STATUS: active
🌡️ SUHU: 47°C
```

---

## 🧬 11. Tambah Rig Baru?

1. Copy folder `xmrig/` ke rig baru
2. Edit:

   * `RIG_NAME` di `automation.sh`
   * Wallet/Thread di `xmrig.sh`
3. Ulangi langkah setup systemd & bot

---
