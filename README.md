# Ubuntu Server 24.04 Security Hardening & CIS Compliance

Repositori otomatisasi Ansible untuk penerapan **Security Hardening** dan **Audit Kepatuhan Keamanan** pada **Ubuntu Server 24.04 LTS (Noble Numbat)** berdasarkan standar **CIS (Center for Internet Security) Benchmark Level 2 Server**.

Proyek ini dilengkapi dengan modul audit otomatis menggunakan **OpenSCAP** dan **CIS-CAT Lite** yang menghasilkan laporan HTML secara berkala untuk memantau peningkatan skor kepatuhan keamanan.

---

## Daftar Isi
- [Fitur Utama](#fitur-utama)
- [Struktur Repositori](#struktur-repositori)
- [Cakupan Hardening OS](#cakupan-hardening-os)
- [Simulasi Web Server: Hardened vs Baseline](#simulasi-web-server-hardened-vs-baseline)
- [Prasyarat Sistem](#prasyarat-sistem)
- [Panduan Penggunaan](#panduan-penggunaan)
  - [1. Konfigurasi Inventaris](#1-konfigurasi-inventaris)
  - [2. Setup Web Server Nginx](#2-setup-web-server-nginx)
  - [3. Menerapkan Hardening OS](#3-menerapkan-hardening-os)
  - [4. Audit Kepatuhan & Verifikasi](#4-audit-kepatuhan--verifikasi)
- [Pengelolaan Laporan Audit](#pengelolaan-laporan-audit)
- [Keamanan & Praktik Terbaik](#keamanan--praktik-terbaik)
- [Lisensi](#lisensi)

---

## Fitur Utama

- **Otomatisasi Penuh**: Menerapkan puluhan parameter keamanan CIS Benchmark Level 2 dalam satu eksekusi playbook Ansible.
- **Audit Ganda (Dual-Engine)**:
  - **OpenSCAP**: Pemindaian XCCDF otomatis menggunakan datastream SCAP Security Guide (SSG) resmi Ubuntu 24.04.
  - **CIS-CAT Lite**: Pemindaian benchmark resmi CIS menggunakan engine Java CLI.
- **Visualisasi Hasil**: Mengunduh laporan evaluasi format HTML langsung ke host pengontrol (`reports/`) dengan symlink otomatis ke laporan terbaru (`*-report-latest.html`).
- **Idempoten & Modular**: Role Ansible disusun modular berdasarkan domain keamanan (Filesystem, Kernel/Sysctl, SSH, PAM/Auth, Firewall/Auditd, dan Remediasi Lanjutan).

---

## Struktur Repositori

```text
├── ansible.cfg                          # Konfigurasi Ansible project
├── apply-hardening.yml                  # Playbook utama penerapan hardening OS
├── setup-webserver.yml                  # Playbook setup Nginx web server (hardened & standard)
├── audit-openscap.yml                    # Playbook audit kepatuhan via OpenSCAP
├── audit-ciscat.yml                      # Playbook audit kepatuhan via CIS-CAT Lite
├── group_vars/
│   ├── hardening_servers.yml            # Variabel profil server hardening (webserver_hardened: true)
│   └── standard_servers.yml             # Variabel profil server baseline (webserver_hardened: false)
├── inventory/
│   ├── hosts.ini.example                # Template inventaris target host
│   └── hosts.ini                        # Inventaris aktif (diabaikan oleh git)
├── roles/
│   ├── nginx_webserver/                 # Role orkestrasi Nginx web server dual-mode
│   │   ├── defaults/main.yml            # Default variabel role
│   │   ├── tasks/
│   │   │   ├── main.yml                 # Orkestrasi percabangan hardened vs standard
│   │   │   ├── hardened.yml             # Task Nginx TLS 1.3, security headers & nftables
│   │   │   └── standard.yml             # Task Nginx baseline standar bawaan
│   │   ├── templates/
│   │   │   ├── hardened_nginx.conf.j2   # Nginx master conf (server_tokens off, DoS buffers)
│   │   │   ├── hardened_security_headers.conf.j2 # Security headers (HSTS, CSP, nosniff, dll)
│   │   │   ├── hardened_site.conf.j2    # Vhost hardened (redirect 301, TLS, block dotfiles)
│   │   │   ├── hardened_index.html.j2   # Dummy dashboard HTML server ter-hardening
│   │   │   ├── standard_default.conf.j2 # Vhost standar baseline bawaan
│   │   │   ├── standard_index.html.j2   # Dummy page HTML server baseline standar
│   │   │   └── nftables.rules.j2        # Ruleset nftables port 22, 80, 443
│   │   └── handlers/main.yml            # Handler reload/restart nginx & nftables
│   └── ubuntu_hardening/                # Role utama hardening OS Ubuntu 24.04
│       ├── defaults/
│       │   └── main.yml                 # Default variables & threshold anti-DDoS / Fail2ban
│       ├── tasks/
│       │   ├── main.yml                 # Orkestrasi task hardening
│       │   ├── filesystem.yml           # Hardening mount options & partisi
│       │   ├── kernel_modules.yml       # Blacklist modul kernel & protokol tidak aman
│       │   ├── sysctl.yml               # Kernel parameters, TCP stack tuning & proteksi DoS
│       │   ├── limits.yml               # File descriptors & socket buffers tuning (nofile)
│       │   ├── services.yml             # Pengelolaan daemon, paket legacy & time sync
│       │   ├── cron.yml                 # Izin & restriksi penjadwal Cron & At
│       │   ├── ssh.yml                  # Hardening konfigurasi OpenSSH Server
│       │   ├── auth.yml                 # PAM, pwquality, faillock, & password aging
│       │   ├── firewall_audit.yml       # Aturan Nftables (rate limiting) & auditd comprehensive
│       │   ├── fail2ban.yml             # Dynamic IP banning & proteksi intrusi via Nftables
│       │   ├── logging.yml              # Systemd-journald & izin berkas /var/log
│       │   ├── system_access.yml        # Sudoers logging, banners, umask & PATH
│       │   └── integrity.yml            # AIDE integrity, bootloader GRUB & AppArmor
│       ├── handlers/
│       │   └── main.yml                 # Handler restart service (sshd, auditd, nftables, fail2ban)
│       ├── templates/
│       │   ├── 99-cis.rules.j2          # Template audit rules CIS
│       │   ├── fail2ban_jail.local.j2   # Konfigurasi jail Fail2ban backend Nftables
│       │   └── nftables.rules.j2        # Template firewall nftables anti-DDoS default-deny
│       └── vars/                        # Variabel spesifik role
├── files/                               # Direktori binary pendukung (misal CIS-CAT Lite zip)
├── reports/                             # Direktori output laporan audit HTML
├── ssg-ubuntu2404-ds.xml                # Datastream SCAP Security Guide Ubuntu 24.04
└── README.md
```

---

## Cakupan Hardening OS

Role `ubuntu_hardening` mengimplementasikan parameter keamanan modular berikut:

1. **Filesystem & Mount Options** (`tasks/filesystem.yml`):
   - Penguatan opsi partisi `/dev/shm` (`defaults,nodev,nosuid,noexec`).
   - Penegakan izin sticky bit pada seluruh direktori *world-writable*.

2. **Blacklist Modul Kernel & Protokol Jaringan** (`tasks/kernel_modules.yml`):
   - Menonaktifkan modul filesystem yang tidak terpakai (cramfs, freevxfs, jffs2, hfs, hfsplus, squashfs, udf, overlay).
   - Menonaktifkan protokol jaringan rentan (dccp, rds, sctp, tipc) serta modul hardware (usb-storage, firewire-core, atm, can).

3. **Kernel Parameters, TCP Tuning & Anti-DDoS** (`tasks/sysctl.yml`):
   - Perlindungan spoofing IP, pengabaian broadcast ICMP, dan penolakan ICMP redirect.
   - Mitigasi serangan SYN flood (`tcp_syncookies = 1`, `tcp_max_syn_backlog = 65535`).
   - Optimalisasi antrean soket & timeout (`somaxconn = 65535`, `tcp_fin_timeout = 15`, `tcp_tw_reuse = 1`).
   - Penguatan tabel conntrack dari saturasi serangan (`nf_conntrack_max = 262144`).
   - Pengaktifan Randomize VA Space (ASLR), dmesg restrict, dan pembatasan ptrace/suid dump.

4. **Batas Sumber Daya Sistem & File Descriptors** (`tasks/limits.yml`):
   - Peningkatan batas file descriptor (`nofile = 65535`) di `/etc/security/limits.d/99-nofile.conf`.
   - Override batas `DefaultLimitNOFILE=65535` pada daemon systemd untuk mencegah error *"Too many open files"* saat lonjakan trafik.

5. **Pengelolaan Layanan & Paket Legacy** (`tasks/services.yml`):
   - Pembersihan paket jaringan insecure (`ftp`, `tnftp`, `telnet`, `inetutils-telnet`, `rsync`).
   - Penonaktifan dan masking daemon tidak terpakai (`rsync`, `apport`, `avahi-daemon`, `cups`).
   - Sinkronisasi waktu otomatis via drop-in NTP `systemd-timesyncd`.
   - Pembatasan Postfix hanya mendengarkan loopback interface.

6. **Penjadwalan Tugas Cron & At** (`tasks/cron.yml`):
   - Kontrol akses ketat berkas `cron.allow` dan `at.allow`, serta penghapusan `.deny`.
   - Pembatasan izin direktori `/etc/cron.*` (0700) dan `/etc/crontab` (0600).

7. **OpenSSH Server Hardening** (`tasks/ssh.yml`):
   - Penonaktifan login `root` via SSH (`PermitRootLogin no`) dan password kosong.
   - Penegakan otentikasi kunci publik (`PubkeyAuthentication yes`, `PasswordAuthentication no`).
   - Pembatasan ciphers, MACs, dan KexAlgorithms ke algoritma kriptografi modern.
   - Konfigurasi banner SSH resmi di `/etc/issue.net`.

8. **Autentikasi & Kebijakan Password** (`tasks/auth.yml`):
   - Kompleksitas kata sandi via `pam_pwquality` (panjang minimal 14, variasi 4 kelas).
   - Penguncian akun setelah percobaan gagal via `pam_faillock`.
   - Pencegahan penggunaan ulang kata sandi lama via `pam_pwhistory`.
   - Timeout sesi shell otomatis (`TMOUT=900`) dan pembatasan perintah `su`.

9. **Firewall Nftables L4 Rate Limiting & Auditd** (`tasks/firewall_audit.yml`):
   - Pengamanan jaringan berbasis **Nftables** dengan aturan default-deny.
   - Drop otomatis terhadap malformed TCP packets dan stealth scans (Xmas, NULL, SYN+FIN).
   - Rate limiting ICMP/Ping flood (`5/s`, burst `10`).
   - Mitigasi SYN Flood berbasis `meter` dinamis (`40/s`, burst `80`) dan concurrent connection cap (`50` per IP).
   - Pengaturan komprehensif **auditd** untuk mencatat modifikasi berkas sensitif dan eksekusi privilese.

10. **Dynamic IP Banning & Intrusion Mitigation** (`tasks/fail2ban.yml`):
    - Integrasi **Fail2ban** dengan backend **Nftables Multiport**.
    - Proteksi otomatis terhadap brute-force SSH (`sshd`).
    - Deteksi & blokir otomatis bagi penyerang yang memicu limit request Nginx HTTP 429 (`nginx-limit-req`).
    - Deteksi otomatis probe scanner dan exploit bot (`nginx-botsearch`).

11. **System Logging & Journald** (`tasks/logging.yml`):
    - Konfigurasi `systemd-journald` (`ForwardToSyslog=yes`, `Storage=persistent`).
    - Pengamanan izin berkas log pada direktori `/var/log`.

12. **Akses Sistem, Banner & Lingkungan Sesi** (`tasks/system_access.yml`):
    - Audit logfile dan session timeout sudo di `/etc/sudoers.d/01-cis-sudo`.
    - Peringatan banner konsol lokal `/etc/issue` & `/etc/motd`, serta pembersihan skrip motd dinamis.
    - Standardisasi default `umask 027` dan sanitasi variabel lingkungan `PATH`.
    - Pembatasan izin berkas inisialisasi interaktif pengguna (dotfiles).

13. **Integritas Sistem, Bootloader & AppArmor** (`tasks/integrity.yml`):
    - Inisialisasi basis data integritas berkas **AIDE** dan monitoring audit tools.
    - Pengamanan argumen kernel bootloader **GRUB** (`audit=1`, `apparmor=1`).
    - Penegakan profil **AppArmor** ke mode enforce (`aa-enforce`).

---

## Simulasi Web Server: Hardened vs Baseline

Agar pengujian keamanan merefleksikan arsitektur server produksi dunia nyata (*real-world web server*) dan bukan sekadar host minimalis dengan port SSH, kedua VM difungsikan sebagai web server aktif menggunakan **Nginx** dengan peran dan parameter keamanan yang berbeda:

| Parameter Evaluasi | `standard-server` (Unhardened Baseline) | `ubuntu-server-2404` (Hardened Production) |
| :--- | :--- | :--- |
| **Profil Mesin** | Node pembanding baseline tanpa hardening | Node produksi ter-hardening CIS Level 2 |
| **Server Tokens / Banner** | **Aktif (`server_tokens on;`)** — mengekspos versi Nginx & OS | **Dihilangkan (`server_tokens off;`)** — hanya menampilkan `nginx` |
| **Protokol HTTP/HTTPS** | Plain HTTP aktif tanpa pengalihan otomatis; HTTPS standar | **HTTP dialihkan paksa (301 Redirect)** ke HTTPS terenkripsi |
| **Validasi Host Header** | Semua host diterima (rentan Host Injection/Open Redirect) | **Tervalidasi ketat** (Host asing ditolak `HTTP 400 Bad Request`) |
| **HSTS (Strict-Transport-Security)** | **Tidak Ada** | **Aktif (`max-age=31536000; includeSubDomains; preload`)** |
| **X-Frame-Options** | **Tidak Terpasang** (Rentan Clickjacking) | **`SAMEORIGIN`** (Terproteksi dari Clickjacking) |
| **X-Content-Type-Options** | **Tidak Terpasang** (MIME Sniffing diizinkan) | **`nosniff`** (Cegah sniffing tipe file) |
| **Content Security Policy (CSP)** | **Tidak Dibatasi** | **Ketat** (`default-src 'self'`, `script-src 'self'`, no unsafe-inline script) |
| **Permissions-Policy** | **Tidak Ada** | **Dibatasi** (`camera=(), microphone=(), geolocation=()`) |
| **Enkripsi TLS & SAN** | Protokol bawaan, sertifikat self-signed tanpa SAN | **TLSv1.2 & TLSv1.3 only**, DHParam 2048-bit, **SAN IP & DNS support** |
| **Mitigasi DoS & Rate Limiting** | Tanpa batasan rate/koneksi | **10 req/s, burst 20, 20 conns/IP (`HTTP 429`)**, timeout ketat |
| **Restriksi Metode HTTP** | Standar Nginx (GET, HEAD) | **GET, HEAD, POST diizinkan** (metode lain ditolak HTTP 405) |
| **Proteksi Dotfiles (`.env`, `.git`)** | Akses file standar (dapat diakses jika ada di disk) | **Diblokir** (Status HTTP 403 Forbidden) |
| **Custom Error Handling** | Error page bawaan Nginx (ekspos versi Nginx) | **Kustom 404 & 50x** (tanpa banner versi/OS atau kebocoran stack trace) |
| **Mitigasi Serangan BREACH** | Gzip compression aktif | **Gzip dinonaktifkan (`gzip off;`)** pada HTTPS |
| **Firewall Host** | Standar / UFW Inactive (Permissive) | **Nftables Stateful Filter** (Default-deny, izin port 22, 80, 443) |
| **Tampilan Halaman Dummy** | Halaman responsif: *Standard Baseline Server* | Halaman responsif: *Hardened Enterprise Dashboard* |

## Prasyarat Sistem

- **Host Pengontrol (Ansible Controller)**:
  - Linux (diuji pada Fedora / Ubuntu)
  - Ansible Core 2.15+ / Python 3.10+
  - OpenSSH Client
- **Target Managed Host**:
  - Ubuntu Server 24.04 LTS (Noble Numbat)
  - Akses `sudo` tanpa password (atau menggunakan `ansible_become_password`)
  - Python 3 terinstal

---

## Panduan Penggunaan

### 1. Konfigurasi Inventaris

Inventaris dibagi menjadi dua kelompok target utama:
- `hardening_servers`: Host yang akan dieksekusi playbook hardening (`ubuntu-server-2404`).
- `standard_servers`: Host standar tanpa hardening dengan flag `hardening_allowed=false`, difungsikan khusus sebagai pembanding baseline untuk pengujian audit OpenSCAP & CIS-CAT (`standard-server`).
- `ubuntu_vms`: Grup induk (children) yang mencakup seluruh target audit kepatuhan.

Salin template inventaris dan sesuaikan alamat IP serta kredensial target VM:

```bash
cp inventory/hosts.ini.example inventory/hosts.ini
nano inventory/hosts.ini
```

Pastikan konektivitas Ansible berjalan dengan baik ke seluruh host:

```bash
ansible all -m ping
```

---

### 2. Setup Web Server Nginx

Jalankan playbook untuk memasang dan mengonfigurasi web server Nginx pada seluruh VM target:
- `standard-server`: Dikonfigurasi dengan parameter standar baseline (unhardened).
- `ubuntu-server-2404`: Dikonfigurasi dengan TLS 1.2/1.3, security headers, dan ruleset nftables terbuka untuk port 80 & 443.

```bash
ansible-playbook setup-webserver.yml
```

#### Verifikasi Respon Web Server & Security Headers

Gunakan `curl` dari controller atau workstation untuk menguji perbedaan keamanan antara kedua node:

```bash
# 1. Uji Baseline Standard Server (Unhardened)
# Respon menampilkan versi Nginx (server_tokens on), plain HTTP diterima, tanpa HSTS
curl -I http://10.10.10.233
curl -k -I https://10.10.10.233

# 2. Uji Hardened Production Server
# HTTP port 80 dialihkan paksa (301) ke HTTPS, server_tokens off, seluruh security headers terpasang
curl -I http://10.10.10.232
curl -k -I https://10.10.10.232

# 3. Uji restriksi metode HTTP terlarang (harus mengembalikan HTTP 405)
curl -k -I -X DELETE https://10.10.10.232

# 4. Uji proteksi file tersembunyi (harus mengembalikan HTTP 403 Forbidden)
curl -k -I https://10.10.10.232/.env

# 5. Uji pencegahan Host Header Injection / Open Redirect (harus ditolak HTTP 400 Bad Request)
curl -H "Host: evil.com" -I http://10.10.10.232
curl -k -H "Host: evil.com" -I https://10.10.10.232

# 6. Uji halaman error 404 kustom ter-hardening (tanpa bocoran banner Nginx)
curl -k -I https://10.10.10.232/nonexistent
```

---

### 3. Menerapkan Hardening OS

Jalankan playbook hardening untuk mengaplikasikan konfigurasi keamanan CIS Benchmark Level 2 pada kelompok `hardening_servers`:

```bash
ansible-playbook apply-hardening.yml
```

> [!NOTE]
> Setelah playbook selesai, sebagian service (seperti SSH, auditd, nftables) akan dimuat ulang secara otomatis. Beberapa parameter kernel atau filesystem mount baru mungkin memerlukan reboot VM target:
> ```bash
> ansible ubuntu_vms -m reboot -b
> ```

---

### 4. Audit Kepatuhan & Verifikasi

Jalankan audit kepatuhan OpenSCAP pada seluruh mesin (`ubuntu_vms`) untuk membandingkan skor antara node baseline standar dengan node yang telah di-hardening:

```bash
# Audit menggunakan OpenSCAP
ansible-playbook audit-openscap.yml
```

Laporan evaluasi HTML per host akan disimpan di folder `reports/`:
- `reports/oscap-report-standard-server-latest.html` (Baseline unhardened)
- `reports/oscap-report-ubuntu-server-2404-latest.html` (Hardened production)

*(Opsional)* Jika menggunakan CIS-CAT Lite, tempatkan file arsip `CIS-CAT-Lite.zip` di dalam folder `files/`, kemudian jalankan:
```bash
ansible-playbook audit-ciscat.yml
```

---

## Pengelolaan Laporan Audit

Laporan audit format HTML yang diunduh dari mesin target disimpan secara lokal di folder `reports/`:
- `reports/oscap-report-<epoch>.html`: Laporan audit OpenSCAP per eksekusi.
- `reports/oscap-report-latest.html`: Symlink otomatis ke laporan OpenSCAP terbaru.
- `reports/ciscat-report-<epoch>.html`: Laporan audit CIS-CAT Lite per eksekusi.
- `reports/ciscat-report-latest.html`: Symlink otomatis ke laporan CIS-CAT terbaru.

Direktori `reports/*.html` secara default diabaikan oleh Git agar tidak memperbesar ukuran repositori.

---

## Keamanan & Praktik Terbaik

- **Jangan commit kredensial**: Berkas `inventory/hosts.ini` berisi informasi kredensial target dan telah dimasukkan ke dalam `.gitignore`. Gunakan Ansible Vault jika ingin mengenkripsi variabel sensitif.
- **Binary CIS-CAT**: File `CIS-CAT-Lite.zip` adalah software berlisensi dari Center for Internet Security dan tidak disertakan dalam commit Git.

---

## Lisensi

Proyek ini didistribusikan di bawah lisensi terbuka [MIT License](LICENSE). Silakan gunakan, modifikasi, dan distribusikan kembali sesuai dengan ketentuan lisensi tersebut.

