<div align="center">

# 🦗 Jangkrik AI — Captive Portal FAS

**WiFi Hotspot Captive Portal berbasis AI**
dibangun di atas **openNDS FAS**, **Flask**, dan **Groq API**
berjalan di router **GL-B1300 (OpenWrt)**

[![Preview Demo](https://img.shields.io/badge/Live_Demo-Preview_Interaktif-d97757?style=for-the-badge)](https://Kendo-id.github.io/Opennds-auth-AI-challenge/preview.html)
[![OpenWrt](https://img.shields.io/badge/OpenWrt-GL--B1300-00b4d8?style=flat-square&logo=openwrt)](https://openwrt.org/)
[![Flask](https://img.shields.io/badge/Flask-2.x-black?style=flat-square&logo=flask)](https://flask.palletsprojects.com/)
[![Python](https://img.shields.io/badge/Python-3.11-blue?style=flat-square&logo=python)](https://python.org/)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

</div>

---

## Daftar Isi

- [Gambaran Umum](#gambaran-umum)
- [Arsitektur Sistem](#arsitektur-sistem)
- [Cara Kerja](#cara-kerja)
- [Fitur Lengkap](#fitur-lengkap)
- [Struktur Direktori](#struktur-direktori)
- [Prasyarat dan Instalasi](#prasyarat-dan-instalasi)
- [Konfigurasi](#konfigurasi)
- [Karakter AI Kendo](#karakter-ai-kendo)
- [API Endpoint](#api-endpoint)
- [Halaman Status Client](#halaman-status-client)
- [Troubleshooting](#troubleshooting)
- [Lisensi](#lisensi)

---

## Gambaran Umum

**Jangkrik AI** adalah sistem captive portal WiFi hotspot yang menggantikan halaman login konvensional dengan pengalaman interaktif berbasis kecerdasan buatan. Setiap pengguna yang terhubung ke jaringan WiFi diarahkan ke portal ini dan diminta menjawab **tantangan kuis AI** sebelum mendapatkan akses internet.

Portal ini terintegrasi penuh dengan daemon **openNDS** (open Network Demarcation System) melalui mekanisme **FAS (Forward Authentication Service)**, sehingga autentikasi bersifat native dan aman tanpa perlu modifikasi kernel.

### Mengapa Jangkrik AI?

| Captive Portal Konvensional | Jangkrik AI |
|---|---|
| Form login username/password | Kuis AI generatif, soal selalu baru |
| Tampilan statis dan membosankan | UI dark mode animatif dengan maskot jangkrik |
| Tidak ada interaksi | Chat asisten AI (Kendo) siap bantu |
| Backend sederhana | Flask + Groq LLM + openNDS FAS |
| Tidak ada status sesi | Halaman status real-time (download/upload/durasi) |

---

## Arsitektur Sistem

```
+------------------------------------------------------------------+
|                        GL-B1300 Router                           |
|                        (OpenWrt 23.x)                            |
|                                                                  |
|  +--------------+    +--------------------------------------+    |
|  |   openNDS    |    |      Kali Linux Chroot (ARM)         |    |
|  |   daemon     |<-->|                                      |    |
|  |              |    |  +-------------+  +--------------+   |    |
|  |  Port 2050   |    |  | portal_app  |  | smarthome_   |   |    |
|  |  (captive)   |    |  | Flask :5001 |  | app :5002    |   |    |
|  +------+-------+    |  +------+------+  +--------------+   |    |
|         |            |         |                             |    |
|         | FAS        |  +------v------+                      |    |
|         +----------->|  | groq_lite   |<-- groq_lite.py      |    |
|                      |  | HTTP wrapper|    (ARM workaround)  |    |
|                      |  +------+------+                      |    |
|                      +---------|----------------------------+     |
|                                |                                  |
|  +-----------------------------v--------------------------------+  |
|  |                    nginx (HTTPS Reverse Proxy)               |  |
|  |     :443 -> portal :5001   |   :8443 -> smarthome :5002     |  |
|  +-------------------------------------------------------------+  |
|                                                                  |
|  +---------------------------------------------------------------+ |
|  |  SD Card (extroot)  /dev/mmcblk0  -- storage + swap 512MB   | |
|  +---------------------------------------------------------------+ |
+------------------------------------------------------------------+
                              |
                    +---------v---------+
                    |    Groq Cloud API  |
                    |  (LLM + TTS STT)  |
                    +-------------------+
```

### Komponen Utama

| Komponen | Deskripsi |
|---|---|
| **openNDS** | Daemon captive portal, mengintersep HTTP client baru |
| **FAS (Forward Auth Service)** | Mekanisme redirect ke Flask untuk autentikasi |
| **portal_app.py** | Flask app utama — quiz, chat AI, autentikasi |
| **groq_lite.py** | Wrapper HTTP Groq API (pengganti SDK yang tidak kompatibel ARM) |
| **client_params.sh** | Shell script openNDS — generate halaman status client |
| **nginx** | Reverse proxy HTTPS, terminator SSL |
| **Kali chroot** | Lingkungan Python/Flask terisolasi di atas OpenWrt |

---

## Cara Kerja

### Alur Autentikasi Lengkap

```
Client terhubung WiFi
        |
        v
openNDS mengintersep HTTP request pertama
        |
        v (redirect FAS)
portal_app.py menerima request dengan parameter:
  - clientip, clientmac, tok (token openNDS)
  - gatewayaddress, b64query
        |
        v
Flask men-generate session per-device (keyed by MAC)
        |
        v
portal.html ditampilkan -> User pilih kategori kuis
        |
        v
/portal/challenge -> Flask request soal ke Groq LLM
        |             (kategori: Umum / Tebak / Religi)
        v
User input jawaban -> POST /portal/verify
        |
        +-- Salah --> feedback + hint, soal tetap
        |
        +-- Benar --> correctCount++
                      |
                      +-- [correctCount >= REQUIRED]
                                |
                                v
                      POST ke openNDS /opennds_auth/
                      dengan token FAS yang valid
                                |
                                v
                      openNDS membuka akses internet
                      client di-redirect ke URL tujuan
```

### Proses FAS (Forward Authentication Service)

openNDS menggunakan mekanisme token terenkripsi (HMAC-SHA256) untuk validasi autentikasi. `portal_app.py` menerima token ini via parameter `b64query`, memvalidasinya, dan setelah kuis selesai mengirimkan konfirmasi autentikasi kembali ke openNDS.

```
openNDS                portal_app.py (Flask)
    |                         |
    |-- FAS redirect -------->| token + clientip + mac
    |                         |-- validasi token
    |                         |-- tampilkan portal.html
    |                         |-- kuis selesai
    |<-- /opennds_auth/ token-| autentikasi client
    |-- buka akses internet   |
    |-- redirect ke URL asal  |
```

---

## Fitur Lengkap

### Sistem Kuis AI

- **Soal generatif** — dihasilkan Groq LLM tiap sesi, tidak ada soal yang berulang
- **3 kategori**: Umum (pengetahuan Indonesia), Tebak (teka-teki), Religi
- **Sistem progress bar** — tampilkan X / N jawaban benar
- **Petunjuk (hint)** otomatis muncul jika jawaban salah
- **Skip soal** tanpa penalti
- **Jumlah soal wajib** dapat dikonfigurasi di `config.py`

### Asisten AI Kendo

- **Persona unik**: karakter *jutek/judes* dengan perilaku *latah*
- Ditenagai Groq LLM (`moonshotai/kimi-k2-instruct` atau model pilihan)
- **Isolasi sesi per-device** — setiap MAC address punya konteks chat sendiri
- **System prompt stabil** — karakter Kendo tidak berubah sepanjang sesi
- Chat terintegrasi langsung di halaman portal

### UI dan Animasi

- **Dark mode** dengan palet warna oleh Anthropic (warm almost-black `#1c1917`)
- **Maskot Jangkrik SVG animatif** — antena bergoyang, kaki bergerak, tangan memegang kopi dengan uap mengepul
- **Topbar blur** — efek glassmorphism dengan `backdrop-filter`
- **Kartu info client** — IP, MAC, gateway, waktu real-time
- **Modal "About"** — slide-up sheet dengan animasi smooth
- **Success overlay** — otomatis transisi ke halaman Status Client

### Halaman Status Client (client_params.sh)

Dirender oleh shell script openNDS, menampilkan:

- Status koneksi (Online badge dengan animasi pulse)
- Waktu mulai dan durasi sesi (real-time counter)
- Penggunaan download/upload sesi ini dalam KB
- Rata-rata kecepatan download/upload dalam Kb/s
- Kuota dan rate limit
- Detail lanjutan (tipe client, interface, gateway address, versi openNDS)
- Tombol Refresh dan Logout

### Keamanan dan Sesi

- Autentikasi token HMAC-SHA256 (native openNDS FAS)
- Sesi per-device diisolasi menggunakan MAC address sebagai key
- HTTPS via nginx reverse proxy (self-signed atau custom cert)
- Tidak ada data user yang disimpan permanen di server

### Kompatibilitas Hardware

- Berjalan di **GL-B1300 (ARM Cortex-A7, 256MB RAM)**
- **groq_lite.py** — implementasi HTTP langsung sebagai pengganti Groq SDK Python yang tidak kompatibel ARM
- Tidak memerlukan systemd — manajemen proses via shell script dengan PID file (`/usr/local/bin/portal`)
  
---

## Struktur Direktori

```
Opennds-auth-AI-challenge/
|
+-- app.py          # Flask app utama (port 5000)
+-- groq_lite.py           # Wrapper HTTP Groq API (ARM workaround)
+-- .env              # Konfigurasi global (API key, REQUIRED, dll)
|+-- templates/
|   +-- portal.html     # UI Portal openNDS
|
+-- portal.html            # Halaman portal kuis + chat Kendo
+-- /etc/opennds/htdoc/
|   +-- splash.css             # CSS untuk halaman status client (openNDS)
+-- /usr/lib/opennds/
|   +-- client_params.sh       # Shell script openNDS status page
|
+-- preview.html           # Preview interaktif untuk GitHub README
|
+-- README.md              # Dokumentasi ini
```
```
---

## Prasyarat dan Instalasi

### Prasyarat

| Kebutuhan | Versi |
|---|---|
| Router | GL-B1300 atau perangkat OpenWrt sejenis |
| OpenWrt | 23.x atau lebih baru |
| openNDS | 10.x |
| Python | 3.11+ (di dalam chroot) |
| Flask | 2.x |
| SD Card | Min. 4 GB (untuk extroot + swap) |
| Groq API Key | [Daftar gratis di console.groq.com](https://console.groq.com/) |

---

### Langkah 1 — Setup Extroot dan Swap (SD Card)

```bash
# Partisi SD card: swap 512MB + ext4 sisanya
fdisk /dev/mmcblk0
# Buat p1 sebagai swap, p2 sebagai ext4

mkswap /dev/mmcblk0p1
mkfs.ext4 /dev/mmcblk0p2

# Mount dan salin overlay ke extroot
mount /dev/mmcblk0p2 /mnt
tar -C /overlay -cvf - . | tar -C /mnt -xf -
umount /mnt

# Konfigurasi fstab untuk mount otomatis
# lalu aktifkan swap
swapon /dev/mmcblk0p1
```

---

### Langkah 2 — Install openNDS

```bash
opkg update
opkg install opennds
```

---

### Langkah 3 — Setup Kali Linux Chroot (ARM)

```bash
# Download rootfs Kali ARM
wget https://kali.download/nethunter-images/current/rootfs/kalifs-arm-full.tar.xz

# Ekstrak ke SD card
mkdir /mnt/kali
tar -xJf kalifs-arm-full.tar.xz -C /mnt/kali

# Buat wrapper script untuk chroot
cat > /usr/local/bin/kali-chroot << 'EOF'
#!/bin/sh
mount --bind /dev  /mnt/kali/dev
mount --bind /proc /mnt/kali/proc
mount --bind /sys  /mnt/kali/sys
chroot /mnt/kali /bin/bash "$@"
EOF
chmod +x /usr/local/bin/kali-chroot
```

---

### Langkah 4 — Install Python dan Dependensi

```bash
kali-chroot

# Di dalam chroot environment:
apt update
apt install -y python3 python3-pip python3-flask nginx

pip install flask requests --break-system-packages
```

---

### Langkah 5 — Clone Repository

```bash
cd /mnt/kali/opt
git clone https://github.com/Kendo-id/Opennds-auth-AI-challenge.git
cd Opennds-auth-AI-challenge
```

---

### Langkah 6 — Konfigurasi Aplikasi

Edit `config.py` sesuai kebutuhan:

```python
GROQ_API_KEY     = "gsk_xxxxxxxxxxxxxxxxxxxx"   # API key dari Groq
REQUIRED_CORRECT = 3                             # Jumlah jawaban benar
PORTAL_HOST      = "0.0.0.0"
PORTAL_PORT      = 5001
SMARTHOME_PORT   = 5002
FAS_KEY          = "your_fas_secret_key"         # Harus sama dengan openNDS config
```

---

### Langkah 7 — Konfigurasi openNDS

Edit `/etc/config/opennds`:

```
config opennds
    option fasport               '5001'
    option fasremoteip           '127.0.0.1'
    option faspath               '/portal/'
    option fas_secure_enabled    '1'
    option faskey                'your_fas_secret_key'
    option gatewayname           'Jangkrik AI Hotspot'
    option maxclients            '20'
    option sessiontimeout        '86400'
    option login_option_enabled  '0'
    option client_params_script  '/opt/Opennds-auth-AI-challenge/client_params.sh'
```

---

### Langkah 8 — Konfigurasi nginx (HTTPS)

Buat sertifikat SSL self-signed:

```bash
mkdir -p /etc/nginx/ssl
openssl req -x509 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/key.pem \
  -out /etc/nginx/ssl/cert.pem \
  -days 365 -nodes \
  -subj "/CN=jangkrik.local"
```

Konfigurasi nginx di `/etc/nginx/sites-enabled/jangkrik`:

```nginx
server {
    listen 443 ssl;
    server_name _;

    ssl_certificate     /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location /portal/ {
        proxy_pass http://127.0.0.1:5001/portal/;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /chat {
        proxy_pass http://127.0.0.1:5001/chat;
    }
}

server {
    listen 8443 ssl;
    server_name _;

    ssl_certificate     /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    location / {
        proxy_pass http://127.0.0.1:5002/;
    }
}
```

---

### Langkah 9 — Script Manajemen Proses

Karena systemd tidak tersedia di lingkungan chroot, digunakan PID file management:

```bash
cat > /usr/local/bin/chatbot << 'EOF'
#!/bin/sh
PIDFILE_PORTAL=/tmp/portal.pid
PIDFILE_SMARTHOME=/tmp/smarthome.pid
APPDIR=/opt/Opennds-auth-AI-challenge

case "$1" in
  start)
    chroot /mnt/kali /bin/bash -c \
      "cd $APPDIR && python3 portal_app.py > /tmp/portal.log 2>&1" &
    echo $! > $PIDFILE_PORTAL
    chroot /mnt/kali /bin/bash -c \
      "cd $APPDIR && python3 smarthome_app.py > /tmp/smarthome.log 2>&1" &
    echo $! > $PIDFILE_SMARTHOME
    echo "Jangkrik AI started."
    ;;
  stop)
    kill $(cat $PIDFILE_PORTAL)    2>/dev/null
    kill $(cat $PIDFILE_SMARTHOME) 2>/dev/null
    rm -f $PIDFILE_PORTAL $PIDFILE_SMARTHOME
    echo "Jangkrik AI stopped."
    ;;
  restart)
    $0 stop && sleep 2 && $0 start
    ;;
  status)
    [ -f $PIDFILE_PORTAL ] \
      && echo "Portal:     running (PID $(cat $PIDFILE_PORTAL))" \
      || echo "Portal:     stopped"
    [ -f $PIDFILE_SMARTHOME ] \
      && echo "Smarthome:  running (PID $(cat $PIDFILE_SMARTHOME))" \
      || echo "Smarthome:  stopped"
    ;;
esac
EOF
chmod +x /usr/local/bin/chatbot

# Jalankan sekarang
chatbot start

# Cek status
chatbot status
```

---

### Langkah 10 — Auto-start saat Boot

Tambahkan ke `/etc/rc.local` sebelum baris `exit 0`:

```bash
# Mount bind untuk chroot
mount --bind /dev  /mnt/kali/dev
mount --bind /proc /mnt/kali/proc
mount --bind /sys  /mnt/kali/sys

# Aktifkan swap
swapon /dev/mmcblk0p1

# Delay lalu jalankan Jangkrik AI
sleep 8 && /usr/local/bin/chatbot start &
```

---

## Konfigurasi

### Variabel Penting di config.py

| Variabel | Default | Deskripsi |
|---|---|---|
| `GROQ_API_KEY` | `""` | API key dari console.groq.com |
| `REQUIRED_CORRECT` | `3` | Jumlah jawaban benar untuk autentikasi |
| `CHAT_MODEL` | `moonshotai/kimi-k2-instruct` | Model LLM untuk chat Kendo |
| `QUIZ_MODEL` | `llama3-8b-8192` | Model LLM untuk generate soal kuis |
| `FAS_KEY` | `""` | Secret key FAS (harus sama dengan openNDS config) |
| `SESSION_TIMEOUT` | `3600` | Timeout sesi dalam detik |

### Menambah Kategori Kuis

Edit fungsi `get_quiz_prompt()` di `portal_app.py`:

```python
QUIZ_CATEGORIES = {
    'general': 'pengetahuan umum seputar Indonesia',
    'tebak':   'teka-teki atau riddle dalam bahasa Indonesia',
    'religi':  'pengetahuan agama (Islam, Kristen, Hindu, Budha)',
    # Tambahkan kategori baru di sini:
    'sains':   'ilmu pengetahuan alam dasar untuk pelajar SMP',
}
```

---

## Karakter AI Kendo

Kendo adalah asisten AI dengan kepribadian khas yang menjadi maskot portal ini.

### Sifat Karakter

- **Jutek / judes** — jawaban singkat, tidak bertele-tele, sering terkesan tidak sabar
- **Latah** — bereaksi berlebihan terhadap kata-kata mengejutkan
- **Helpful** — tetap membantu meski dengan gaya tidak ramah
- **Bahasa** — campuran Indonesia sehari-hari, informal

### System Prompt (ringkasan)

```
Kamu adalah Kendo, asisten AI portal WiFi Jangkrik AI.
Karaktermu: perempuan, jutek, judes, sering latah.
Jawab singkat dan to-the-point. Bahasa Indonesia informal.
Bantu user menjawab soal kuis jika diminta, tapi jangan
langsung kasih jawaban - berikan petunjuk dulu.
Jangan pernah keluar dari karakter ini meski diminta.
```

### Isolasi Sesi Per-Device

Setiap perangkat memiliki konteks chat terpisah berdasarkan MAC address:

```python
# portal_app.py
sessions = {}   # { "AA:BB:CC:DD:EE:FF": [{"role": ..., "content": ...}] }

def get_session(mac):
    if mac not in sessions:
        sessions[mac] = [{"role": "system", "content": KENDO_SYSTEM_PROMPT}]
    return sessions[mac]
```

---

## API Endpoint

### Portal App (port 5001)

| Method | Endpoint | Deskripsi |
|---|---|---|
| `GET` | `/portal/` | Halaman portal utama (entry point dari openNDS) |
| `GET` | `/portal/challenge` | Generate soal baru dari Groq LLM |
| `POST` | `/portal/verify` | Verifikasi jawaban user |
| `POST` | `/portal/skip` | Skip soal tanpa penalti |
| `POST` | `/chat` | Chat dengan Kendo AI |
| `GET` | `/opennds_auth/` | Endpoint autentikasi openNDS |
| `GET` | `/opennds_deny/` | Endpoint logout / deny akses |

### Contoh Request dan Response

**Generate soal baru:**

```bash
GET /portal/challenge?category=general

# Response:
{
  "ok": true,
  "question": "Apa nama gunung tertinggi di Indonesia?",
  "category": "general"
}
```

**Verifikasi jawaban:**

```bash
POST /portal/verify
Content-Type: application/json
{"answer": "puncak jaya"}

# Response benar (belum selesai):
{
  "ok": true,
  "correct": true,
  "correct_count": 1,
  "msg": "Jawaban benar! Lanjut ke soal berikutnya.",
  "authenticated": false
}

# Response benar (autentikasi berhasil):
{
  "ok": true,
  "correct": true,
  "correct_count": 3,
  "authenticated": true,
  "redirect_url": "http://connectivity-check.example.com"
}

# Response salah:
{
  "ok": true,
  "correct": false,
  "msg": "Belum tepat, coba lagi!",
  "hint": "Gunung ini ada di Papua."
}
```

---

## Halaman Status Client

Halaman status dirender oleh `client_params.sh` dan dipanggil openNDS setelah autentikasi berhasil. File ini adalah shell script yang mengeksekusi perintah `ndsctl` dan menghasilkan HTML lengkap dengan CSS dari `splash.css`.

### Parameter yang Ditampilkan

| Parameter | Sumber Data |
|---|---|
| IP Address | `ndsctl json <clientip>` — field `ip` |
| MAC Address | `ndsctl json <clientip>` — field `mac` |
| Waktu mulai sesi | `session_start` (epoch) dikonversi dengan `date` |
| Durasi sesi | Dihitung dari `session_start` sampai waktu sekarang |
| Download / Upload sesi | `download_this_session` / `upload_this_session` dalam KB |
| Avg kecepatan | `download_session_avg` / `upload_session_avg` dalam Kb/s |
| Kuota | `download_quota` / `upload_quota` |
| Interface | `clientif` diparse menjadi LocalZone atau MeshZone |
| Versi NDS | `version` dari ndsctl json |

### Mode Halaman

Script mendukung dua mode yang ditentukan oleh parameter `$status`:

- **`status`** — tampilkan informasi sesi lengkap (mode normal setelah login)
- **`err511`** — tampilkan tombol redirect ke portal (mode error HTTP 511)

---

## Troubleshooting

### Flask tidak bisa akses Groq API di ARM

Groq SDK Python resmi tidak kompatibel dengan ARM Cortex-A9. Solusi: gunakan `groq_lite.py` yang sudah tersedia di repo ini.

```python
# Ganti:
from groq import Groq
client = Groq(api_key=KEY)
response = client.chat.completions.create(...)

# Dengan:
from groq_lite import GroqLite
client = GroqLite(api_key=KEY)
response = client.chat(model=MODEL, messages=MESSAGES)
```

### openNDS tidak redirect ke portal

Periksa apakah Flask sudah berjalan dan port benar:

```bash
netstat -tlnp | grep 5001
curl -v http://127.0.0.1:5001/portal/
logread | grep opennds
```

### Qwen3 menampilkan tag `<think>` di response

Tambahkan fungsi strip berikut dan aplikasikan pada semua response AI:

```python
import re

def strip_think(text):
    return re.sub(r'<think>.*?</think>', '', text, flags=re.DOTALL).strip()

# Penggunaan:
reply = strip_think(response_from_groq)
```

### Sesi chat tercampur antar user

Pastikan MAC address digunakan sebagai session key, bukan IP address:

```python
# BENAR - MAC address unik per perangkat
session_key = request.args.get('mac', 'unknown')

# SALAH - IP bisa berubah atau shared NAT
session_key = request.remote_addr
```

### nginx HTTPS gagal koneksi

Pastikan sertifikat sudah dibuat dan path-nya benar:

```bash
ls -la /etc/nginx/ssl/
nginx -t   # test konfigurasi
nginx -s reload
```

### Memori router habis (OOM)

Aktifkan swap dan monitor penggunaan memori:

```bash
swapon /dev/mmcblk0p1
free -h
top   # monitor proses yang boros memori
```

### Script client_params.sh tidak bisa akses ndsctl

Pastikan openNDS sudah berjalan dan path ndsctl benar:

```bash
/etc/init.d/opennds status
which ndsctl
ndsctl status
```

---

## Catatan Teknis

**Pure ASCII untuk JavaScript** — Semua komentar dan string di dalam tag `<script>` harus menggunakan karakter ASCII saja. Karakter Unicode di komentar JS menyebabkan script parse error di beberapa embedded WebView dan tidak kompatibel dengan ArduinoDroid.

**Tanpa systemd** — Lingkungan chroot OpenWrt tidak memiliki systemd. Manajemen proses dilakukan manual via shell script dengan PID file di `/tmp/`.

**groq_lite.py wajib di ARM** — Groq Python SDK resmi menggunakan dependensi yang tidak tersedia di ARM Cortex-A9. `groq_lite.py` mengimplementasi ulang HTTP call secara langsung menggunakan `requests`.

**Web Speech API dan localhost** — Jika menggunakan fitur STT/TTS berbasis Web Speech API, pastikan akses dilakukan via `localhost` bukan `127.0.0.1`. Browser modern membatasi API ini hanya pada origin yang dianggap secure.

**Extroot wajib** — OpenWrt default hanya memiliki storage NAND yang sangat terbatas. Extroot ke SD card wajib untuk menampung Kali chroot dan dependensi Python.

---

## Lisensi

Proyek ini dirilis di bawah lisensi **MIT**.

`client_params.sh` diadaptasi dari kode BlueWave Projects and Services &copy; 2015-2022, dirilis di bawah **GNU GPL**.

---

<div align="center">

Dibuat dengan ❤️ oleh **[Kendo-id](https://github.com/Kendo-id)**

🦗 *Portal WiFi bukan harus membosankan*

</div>
