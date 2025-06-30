### **Tutorial Setup Ubuntu Server untuk Mining Monero (XMRig) dengan Monitoring Otomatis**

#### **Persyaratan:**
- Ubuntu Server 20.04/22.04 LTS (fresh install)
- Akses root/sudo
- Koneksi internet stabil

---

### **Langkah 1: Persiapan Awal**
Update sistem dan install paket dasar:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git build-essential cmake libuv1-dev libssl-dev libhwloc-dev
```

---

### **Langkah 2: Install XMRig**
1. Clone repository XMRig:
```bash
git clone https://github.com/xmrig/xmrig.git
cd xmrig
```

2. Build XMRig:
```bash
mkdir build
cd build
cmake ..
make -j$(nproc)
```

---

### **Langkah 3: Konfigurasi XMRig**
1. Buat file konfigurasi `xmrig.sh`:
```bash
cat <<EOF | sudo tee /usr/local/bin/xmrig.sh
#!/bin/bash
sudo modprobe msr

./xmrig \\
  -o pool.supportxmr.com:3333 \\
  -u 42L5wvMMffV29yRRR6ojzX363dWaqtRS1Rpx9hP9PdnidhRtXkikZdvRJuqx4kov3n7YX6ZFa1yA2JhGGinP745g6MmsUEC \\
  -p \$HOSTNAME \\
  --donate-level 1 \\
  --algo rx/0 \\
  --threads=\$(nproc) \\
  --no-color \\
  --huge-pages \\
  --randomx-1gb-pages \\
  --asm=auto \\
  --log-file=/var/log/xmrig.log \\
  --verbose
EOF

sudo chmod +x /usr/local/bin/xmrig.sh
```

---

### **Langkah 4: Buat Systemd Service untuk XMRig**
```bash
cat <<EOF | sudo tee /etc/systemd/system/xmrig.service
[Unit]
Description=XMRig Miner Service
After=network.target

[Service]
Type=simple
ExecStart=/home/goppe/xmrig/build/xmrig.sh
WorkingDirectory=/home/goppe/xmrig/build
Restart=always
Nice=10
CPUWeight=90
MemoryMax=6G
LimitMEMLOCK=infinity
LimitNOFILE=65535
RestartSec=10
User=root
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target

EOF

sudo systemctl daemon-reload
sudo systemctl enable xmrig
```

---

### **Langkah 5: Install Dependencies Monitoring**
```bash
sudo apt install -y jq lm-sensors bc curl
```

---

### **Langkah 6: Setup Script Monitoring**
1. Buat file script:
```bash
cat <<EOF | sudo tee /usr/local/bin/automation.sh
#!/bin/bash
# === KONFIGURASI ===
TOKEN="Token Telegram"
CHAT_ID="Isi Punya kalian"
RIG_NAME="\$(hostname)"
# ... (paste seluruh isi script newupgradeautomation.sh di sini) ...
EOF

sudo chmod +x /usr/local/bin/automation.sh
```

2. Buat service untuk monitoring:
```bash
cat <<EOF | sudo tee /etc/systemd/system/automation.service
[Unit]
Description=Mining Automation Service
After=network.target

[Service]
ExecStart=/usr/bin/bash /home/goppe/xmrig/build/automation.sh
WorkingDirectory=/home/goppe/xmrig/build
Restart=always
RestartSec=10
User=root
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable automation
```

---

### **Langkah 7: Konfigurasi Logging**
1. Buat direktori log:
```bash
sudo mkdir -p /var/lib/rig_monitor
sudo touch /var/log/{xmrig,rig_monitor}.log
sudo chmod 644 /var/log/{xmrig,rig_monitor}.log
```

2. Rotasi log otomatis:
```bash
cat <<EOF | sudo tee /etc/logrotate.d/mining
/var/log/xmrig.log {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 644 root root
}

/var/log/rig_monitor.log {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 644 root root
}
EOF
```

---

### **Langkah 8: Mulai Semua Layanan**
```bash
sudo systemctl start automation
```

---

### **Langkah 9: Verifikasi**
1. Cek status mining:
```bash
sudo systemctl status xmrig
```

2. Cek status monitoring:
```bash
sudo systemctl status automation
```

3. Lihat log real-time:
```bash
journalctl -fu xmrig
journalctl -fu automation
```

---

### **Fitur Telegram Bot**
Kirim perintah berikut ke bot Telegram Anda:
- `/cek RIG` - Status rig saat ini
- `/laporan` - Laporan accepted shares
- `/hapus laporan` - Reset data shares

---

### **Tips Optimasi:**
1. Aktifkan Huge Pages:
```bash
echo "vm.nr_hugepages = 1280" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

2. Setel governor ke performance:
```bash
sudo apt install -y cpufrequtils
echo 'GOVERNOR="performance"' | sudo tee /etc/default/cpufrequtils
sudo systemctl disable ondemand
```

3. Untuk sistem headless:
```bash
sudo systemctl set-default multi-user.target
```

---

Dengan tutorial ini, Ubuntu Server Anda akan:
✅ Auto-start mining saat boot  
✅ Monitoring otomatis suhu & hashrate  
✅ Laporan real-time ke Telegram  
✅ Rotasi log otomatis  
✅ Optimasi dasar untuk mining  

Untuk update script di masa depan, cukup edit `/usr/local/bin/rig_monitor.sh` dan restart service:
```bash
sudo systemctl restart automation
```
