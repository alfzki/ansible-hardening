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
│       │   ├── sysctl.yml               # Kernel parameters & perlindungan jaringan
│       │   ├── ssh.yml                  # Hardening konfigurasi OpenSSH Server
│       │   ├── auth.yml                 # PAM, pwquality, faillock, & password aging
│       │   ├── firewall_audit.yml       # Aturan Nftables & auditd comprehensive
│       │   ├── cis_remediation.yml      # Remediasi CIS: sanitasi, cron, services
│       │   └── cis_remediation_v2.yml   # Remediasi CIS: AIDE, GRUB, journald, AppArmor
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

Role `ubuntu_hardening` mengimplementasikan parameter keamanan berikut:

1. **Filesystem & Mount Options** (`tasks/filesystem.yml`):
   - Menonaktifkan modul filesystem yang tidak terpakai (cramfs, freevxfs, jffs2, hfs, hfsplus, squashfs, udf).
   - Penguatan opsi partisi `/tmp`, `/var/tmp`, `/dev/shm` (`nodev`, `nosuid`, `noexec`).

2. **Kernel Parameters & Sysctl** (`tasks/sysctl.yml`):
   - Perlindungan spoofing IP, pengabaian broadcast ICMP, dan penolakan ICMP redirect.
   - Mitigasi serangan SYN flood (`tcp_syncookies = 1`).
   - Pengaktifan Randomize VA Space (ASLR) dan pembatasan ptrace scope.

3. **OpenSSH Server Hardening** (`tasks/ssh.yml`):
   - Penonaktifan login `root` langsung via SSH (`PermitRootLogin no`).
   - Penonaktifan otentikasi password kosong (`PermitEmptyPasswords no`).
   - Pembatasan ciphers dan MACs ke algoritma kriptografi modern.
   - Konfigurasi `ClientAliveInterval` dan batas percobaan login (`MaxAuthTries`).

4. **Autentikasi & Kebijakan Password** (`tasks/auth.yml`):
   - Kompleksitas kata sandi via `pam_pwquality` (panjang minimal, variasi karakter).
   - Penguncian akun setelah percobaan gagal via `pam_faillock`.
   - Pencegahan penggunaan ulang kata sandi lama via `pam_pwhistory`.
   - Timeout sesi shell otomatis (`TMOUT=900`) dan pembatasan perintah `su`.

5. **Firewall & Logging/Audit** (`tasks/firewall_audit.yml`):
   - Pengamanan jaringan berbasis **Nftables** dengan aturan default-deny.
   - Pengaturan komprehensif **auditd** untuk mencatat modifikasi file sensitif, eksekusi privilese, serta perubahan konfigurasi sistem.

6. **Remediasi Lanjutan CIS** (`tasks/cis_remediation.yml` & `tasks/cis_remediation_v2.yml`):
   - Pengamanan hak akses direktori/file cron dan sistem.
   - Penonaktifan layanan lama/insecure (rpcbind, nis, rsh, talk, telnet, tftp, dll).
   - Inisialisasi dan verifikasi integritas berkas menggunakan **AIDE**.
   - Pengamanan bootloader **GRUB** dengan kata sandi.
   - Konfigurasi penyimpanan aman log **systemd-journald**.
   - Penegakan profil keamanan **AppArmor** (`enforce mode`).

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
