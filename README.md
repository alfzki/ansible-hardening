# Ubuntu Server 24.04 Security Hardening & CIS Compliance

Repositori otomatisasi Ansible untuk penerapan **Security Hardening** dan **Audit Kepatuhan Keamanan** pada **Ubuntu Server 24.04 LTS (Noble Numbat)** berdasarkan standar **CIS (Center for Internet Security) Benchmark Level 2 Server**.

Proyek ini dilengkapi dengan modul audit otomatis menggunakan **OpenSCAP** dan **CIS-CAT Lite** yang menghasilkan laporan HTML secara berkala untuk memantau peningkatan skor kepatuhan keamanan.

---

## Daftar Isi
- [Fitur Utama](#fitur-utama)
- [Struktur Repositori](#struktur-repositori)
- [Cakupan Hardening](#cakupan-hardening)
- [Prasyarat Sistem](#prasyarat-sistem)
- [Panduan Penggunaan](#panduan-penggunaan)
  - [1. Konfigurasi Inventaris](#1-konfigurasi-inventaris)
  - [2. Audit Kepatuhan Awal (Baseline)](#2-audit-kepatuhan-awal-baseline)
  - [3. Menerapkan Hardening](#3-menerapkan-hardening)
  - [4. Audit Kepatuhan Akhir & Verifikasi](#4-audit-kepatuhan-akhir--verifikasi)
- [Pengelolaan Laporan Audit](#pengelolaan-laporan-audit)
- [Keamanan & Praktik Terbaik](#keamanan--praktik-terbaik)

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
├── apply-hardening.yml                  # Playbook utama penerapan hardening
├── audit-openscap.yml                    # Playbook audit kepatuhan via OpenSCAP
├── audit-ciscat.yml                      # Playbook audit kepatuhan via CIS-CAT Lite
├── inventory/
│   ├── hosts.ini.example                # Template inventaris target host
│   └── hosts.ini                        # Inventaris aktif (diabaikan oleh git)
├── roles/
│   └── ubuntu_hardening/                # Role utama hardening Ubuntu 24.04
│       ├── tasks/
│       │   ├── main.yml                 # Orkestrasi task hardening
│       │   ├── filesystem.yml           # Hardening mount options & partisi
│       │   ├── kernel_modules.yml       # Blacklist modul kernel & protokol tidak aman
│       │   ├── sysctl.yml               # Kernel parameters & perlindungan jaringan
│       │   ├── services.yml             # Pengelolaan daemon, paket legacy & time sync
│       │   ├── cron.yml                 # Izin & restriksi penjadwal Cron & At
│       │   ├── ssh.yml                  # Hardening konfigurasi OpenSSH Server
│       │   ├── auth.yml                 # PAM, pwquality, faillock, & password aging
│       │   ├── firewall_audit.yml       # Aturan Nftables & auditd comprehensive
│       │   ├── logging.yml              # Systemd-journald & izin berkas /var/log
│       │   ├── system_access.yml        # Sudoers logging, banners, umask & PATH
│       │   └── integrity.yml            # AIDE integrity, bootloader GRUB & AppArmor
│       ├── handlers/
│       │   └── main.yml                 # Handler restart service (sshd, auditd, nftables)
│       ├── templates/
│       │   ├── 99-cis.rules.j2          # Template audit rules CIS
│       │   └── nftables.rules.j2        # Template firewall nftables default-deny
│       └── vars/                        # Variabel spesifik role
├── files/                               # Direktori binary pendukung (misal CIS-CAT Lite zip)
├── reports/                             # Direktori output laporan audit HTML
├── ssg-ubuntu2404-ds.xml                # Datastream SCAP Security Guide Ubuntu 24.04
└── README.md
```

---

## Cakupan Hardening

Role `ubuntu_hardening` mengimplementasikan parameter keamanan modular berikut:

1. **Filesystem & Mount Options** (`tasks/filesystem.yml`):
   - Penguatan opsi partisi `/dev/shm` (`defaults,nodev,nosuid,noexec`).
   - Penegakan izin sticky bit pada seluruh direktori *world-writable*.

2. **Blacklist Modul Kernel & Protokol Jaringan** (`tasks/kernel_modules.yml`):
   - Menonaktifkan modul filesystem yang tidak terpakai (cramfs, freevxfs, jffs2, hfs, hfsplus, squashfs, udf, overlay).
   - Menonaktifkan protokol jaringan rentan (dccp, rds, sctp, tipc) serta modul hardware (usb-storage, firewire-core, atm, can).

3. **Kernel Parameters & Sysctl** (`tasks/sysctl.yml`):
   - Perlindungan spoofing IP, pengabaian broadcast ICMP, dan penolakan ICMP redirect.
   - Mitigasi serangan SYN flood (`tcp_syncookies = 1`).
   - Pengaktifan Randomize VA Space (ASLR), dmesg restrict, dan pembatasan ptrace/suid dump.

4. **Pengelolaan Layanan & Paket Legacy** (`tasks/services.yml`):
   - Pembersihan paket jaringan insecure (`ftp`, `tnftp`, `telnet`, `inetutils-telnet`, `rsync`).
   - Penonaktifan dan masking daemon tidak terpakai (`rsync`, `apport`, `avahi-daemon`, `cups`).
   - Sinkronisasi waktu otomatis via drop-in NTP `systemd-timesyncd`.
   - Pembatasan Postfix hanya mendengarkan loopback interface.

5. **Penjadwalan Tugas Cron & At** (`tasks/cron.yml`):
   - Kontrol akses ketat berkas `cron.allow` dan `at.allow`, serta penghapusan `.deny`.
   - Pembatasan izin direktori `/etc/cron.*` (0700) dan `/etc/crontab` (0600).

6. **OpenSSH Server Hardening** (`tasks/ssh.yml`):
   - Penonaktifan login `root` via SSH (`PermitRootLogin no`) dan password kosong.
   - Penegakan otentikasi kunci publik (`PubkeyAuthentication yes`, `PasswordAuthentication no`).
   - Pembatasan ciphers, MACs, dan KexAlgorithms ke algoritma kriptografi modern.
   - Konfigurasi banner SSH resmi di `/etc/issue.net`.

7. **Autentikasi & Kebijakan Password** (`tasks/auth.yml`):
   - Kompleksitas kata sandi via `pam_pwquality` (panjang minimal 14, variasi 4 kelas).
   - Penguncian akun setelah percobaan gagal via `pam_faillock`.
   - Pencegahan penggunaan ulang kata sandi lama via `pam_pwhistory`.
   - Timeout sesi shell otomatis (`TMOUT=900`) dan pembatasan perintah `su`.

8. **Firewall & Aturan Auditd** (`tasks/firewall_audit.yml`):
   - Pengamanan jaringan berbasis **Nftables** dengan aturan default-deny.
   - Pengaturan komprehensif **auditd** untuk mencatat modifikasi berkas sensitif, eksekusi privilese, dan perubahan konfigurasi sistem.

9. **System Logging & Journald** (`tasks/logging.yml`):
   - Konfigurasi `systemd-journald` (`ForwardToSyslog=yes`, `Storage=persistent`).
   - Pengamanan izin berkas log pada direktori `/var/log`.

10. **Akses Sistem, Banner & Lingkungan Sesi** (`tasks/system_access.yml`):
    - Audit logfile dan session timeout sudo di `/etc/sudoers.d/01-cis-sudo`.
    - Peringatan banner konsol lokal `/etc/issue` & `/etc/motd`, serta pembersihan skrip motd dinamis.
    - Standardisasi default `umask 027` dan sanitasi variabel lingkungan `PATH`.
    - Pembatasan izin berkas inisialisasi interaktif pengguna (dotfiles).

11. **Integritas Sistem, Bootloader & AppArmor** (`tasks/integrity.yml`):
    - Inisialisasi basis data integritas berkas **AIDE** dan monitoring audit tools.
    - Pengamanan argumen kernel bootloader **GRUB** (`audit=1`, `apparmor=1`).
    - Penegakan profil **AppArmor** ke mode enforce (`aa-enforce`).

---

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

Salin template inventaris dan sesuaikan alamat IP serta kredensial target VM:

```bash
cp inventory/hosts.ini.example inventory/hosts.ini
nano inventory/hosts.ini
```

Pastikan konektivitas Ansible berjalan dengan baik:

```bash
ansible all -m ping
```

---

### 2. Audit Kepatuhan Awal (Baseline)

Sebelum menerapkan hardening, jalankan audit awal untuk mengetahui skor kepatuhan saat ini:

```bash
# Audit menggunakan OpenSCAP
ansible-playbook audit-openscap.yml
```

Laporan HTML awal akan disimpan di direktori `reports/` dan dapat dilihat di browser:
```bash
xdg-open reports/oscap-report-latest.html
```

*(Opsional)* Jika menggunakan CIS-CAT Lite, tempatkan file arsip `CIS-CAT-Lite.zip` di dalam folder `files/`, kemudian jalankan:
```bash
ansible-playbook audit-ciscat.yml
```

---

### 3. Menerapkan Hardening

Jalankan playbook hardening untuk mengaplikasikan konfigurasi keamanan:

```bash
ansible-playbook apply-hardening.yml
```

> [!NOTE]
> Setelah playbook selesai, sebagian service (seperti SSH, auditd, nftables) akan dimuat ulang secara otomatis. Beberapa parameter kernel atau filesystem mount baru mungkin memerlukan reboot VM target:
> ```bash
> ansible ubuntu_vms -m reboot -b
> ```

---

### 4. Audit Kepatuhan Akhir & Verifikasi

Jalankan kembali pemindaian audit untuk memverifikasi peningkatan skor:

```bash
ansible-playbook audit-openscap.yml
```

Bandingkan skor hasil pemindaian awal dengan laporan terbaru di `reports/oscap-report-latest.html`.

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
