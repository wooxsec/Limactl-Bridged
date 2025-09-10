# Lima (limactl) — Bridged Networking di macOS (step‑by‑step)
## 🎥 Demo Tutorial

<video src="lima.mp4" controls width="700"></video>

Untuk membuat instance Lima (`limactl`) mendapatkan IP sendiri di LAN menggunakan **`socket_vmnet`**. Tutorial ini merangkum langkah yang saya lakukan (install, konfigurasi, troubleshooting)

---

## Ringkasan

Tujuan: agar VM Lima tampil sebagai perangkat real di jaringan lokal (mendapat IP seperti `192.168.8.x`), bukan hanya lewat NAT/port‑forward dari host.

Komponen utama:

* `lima` / `limactl`
* `socket_vmnet` (helper yang mengizinkan bridged networking di macOS)
* file `networks.yaml` Lima (global) dan konfigurasi instance (`lima.yaml` / `limactl edit <instance>`)

> Catatan: contoh perintah dituliskan untuk macOS Apple Silicon (Homebrew di `/opt/homebrew`). Untuk Intel Mac, ganti `/opt/homebrew` dengan `/usr/local`.

---

## Disclaimer & keamanan

* Bridging membuat VM terlihat di jaringan lokal — artinya VM bisa di-scan/dihubungi perangkat lain. Hati‑hati jika menjalankan layanan sensitif.
* Perubahan `sudoers` diperlukan untuk Lima agar dapat menjalankan helper jaringan sebagai root. Pastikan kamu memahami implikasinya.

---

## Prasyarat

* macOS dengan `lima` (instal via `brew install lima`) sudah terpasang.
* Homebrew terpasang.
* Akses `sudo` di host (Administrator).

---

## Langkah 1 — Pasang `socket_vmnet`

```bash
brew install socket_vmnet
```

Catatan output Homebrew: `socket_vmnet` bersifat *keg-only* dan membutuhkan hak root saat dijalankan. Lokasi utama pada Apple Silicon biasanya:

```
/opt/homebrew/Cellar/socket_vmnet/<version>/bin/socket_vmnet
/opt/homebrew/opt/socket_vmnet/bin/socket_vmnet  # ini biasanya symlink ke Cellar
```

Untuk Intel Mac path-nya kemungkinan di `/usr/local/Cellar/...`.

---

## Langkah 2 — Pastikan Lima tidak mereferensi symlink

Lima menolak `paths.socketVMNet` yang menunjuk ke *symlink*. Dia ingin path ke binary asli (bukan symlink). Cari lokasi binari asli:

```bash
ls -l /opt/homebrew/opt/socket_vmnet/bin/socket_vmnet
# atau
ls -l /opt/homebrew/Cellar/socket_vmnet/*/bin/socket_vmnet
```

Ambil path yang mengarah ke `.../Cellar/.../bin/socket_vmnet` (mis. `/opt/homebrew/Cellar/socket_vmnet/1.2.1/bin/socket_vmnet`).

---

## Langkah 3 — Edit `networks.yaml` global Lima

File global yang memuat setelan jaringan biasanya ada di salah satu lokasi:

* `/opt/homebrew/share/lima/networks.yaml`  (Apple Silicon)
* `/usr/local/share/lima/networks.yaml`      (Intel)

Buka file tersebut (gunakan `sudo` jika perlu) dan ubah/isi menjadi:

```yaml
paths:
  socketVMNet: /opt/socket_vmnet/bin/socket_vmnet

networks:
  bridged:
    mode: bridged
    interface: en0   # ganti sesuai interface lan (cek ifconfig)
```

> Penting: `socketVMNet` harus menunjuk langsung ke binary di `Cellar` — bukan ke `/opt/homebrew/opt/...` jika itu symlink.

---

## Langkah 4 — (Alternatif) Copy binary ke `/opt/socket_vmnet/bin`

Jika kamu lebih suka supaya path yang dicari Lima ada di `/opt/socket_vmnet/bin/socket_vmnet`, salin binary asli ke situ (lebih tahan terhadap masalah symlink):

```bash
sudo mkdir -p /opt/socket_vmnet/bin
sudo cp /opt/homebrew/Cellar/socket_vmnet/1.2.1/bin/socket_vmnet /opt/socket_vmnet/bin/
sudo chmod 755 /opt/socket_vmnet/bin/socket_vmnet
```

Setelah ini `networks.yaml` dapat menunjuk ke `/opt/socket_vmnet/bin/socket_vmnet`.

---

## Langkah 5 — Pasang aturan sudoers untuk Lima

Lima memerlukan hak khusus sudo untuk menjalankan helper jaringan. Jalankan perintah ini untuk menghasilkan file sudoers yang aman dan menginstalnya:

```bash
limactl sudoers > etc_sudoers.d_lima
sudo install -o root etc_sudoers.d_lima /private/etc/sudoers.d/lima
sudo chmod 440 /private/etc/sudoers.d/lima
```

Cek isinya:

```bash
sudo cat /private/etc/sudoers.d/lima
```

---

## Langkah 6 — Edit konfigurasi instance Lima (bridged)

Edit instance yang kamu gunakan (misal `default`):

```bash
EDITOR=nano limactl edit default
```

Tambahkan bagian `networks` pada `lima.yaml` instance (contoh minimal):

```yaml
networks:
  - lima: bridged
    interface: en0
```

Simpan, keluar, lalu start instance:

```bash
limactl stop default
limactl start default
```

Jika instance belum pernah dibuat, kamu bisa `limactl create` atau `limactl start debian` dll.

---

## Langkah 7 — Verifikasi & tes konektivitas

Masuk ke VM:

```bash
limactl shell default
# di dalam VM
ip addr show
ping 8.8.8.8
```

Di host (macOS) cek interface dan IP:

```bash
ifconfig | grep inet
# cek apakah ada IP VM di subnet yang sama, atau lihat interface bridge seperti `bridge100`
```

Dari VM coba ping perangkat di LAN (contoh router):

```bash
ping 192.168.8.1
```

Dari host coba ping IP VM — kalau VM sudah di-subnet yang sama, seharusnya berhasil:

```bash
ping <IP_VM>
```

---

## Troubleshooting umum

### 1) `networks.yaml: "/opt/socket_vmnet/bin/socket_vmnet" is a symlink`

* Lima menolak symlink untuk `paths.socketVMNet`. Solusi:

  * Ubah `networks.yaml` supaya menunjuk ke path asli di `Cellar`, atau
  * Copy binary ke `/opt/socket_vmnet/bin/socket_vmnet` (bukan symlink).

### 2) `can't read "/private/etc/sudoers.d/lima"` atau error terkait sudoers

* Jalankan:

  ```bash
  limactl sudoers > etc_sudoers.d_lima
  sudo install -o root etc_sudoers.d_lima /private/etc/sudoers.d/lima
  sudo chmod 440 /private/etc/sudoers.d/lima
  ```

### 3) Host tidak bisa ping IP VM (VM di subnet 192.168.5.x)

* Cek apakah ada interface bridge di host (`ifconfig`) yang menggunakan subnet 192.168.5.x (mis. `bridge100`);
* Jika tidak, mungkin bridged belum berjalan sepenuhnya atau VM mendapat subnet berbeda dari host. Pastikan `interface: en0` benar.
* Workaround: gunakan `portForwards` di `lima.yaml` untuk expose layanan (mis. SSH) ke host.

### 4) Perbedaan path Apple Silicon vs Intel

* Apple Silicon (M1/M2/M3): Homebrew default ke `/opt/homebrew/...` dan `Cellar` di `/opt/homebrew/Cellar`.
* Intel: Homebrew biasanya di `/usr/local/...`.

---

## Opsi tambahan

* **Jadikan socket\_vmnet service otomatis** (jika mau selalu aktif):

  ```bash
  sudo brew services start socket_vmnet
  ```

* **Set IP statis di VM** (Ubuntu/Debian dengan `netplan`):
  Edit `/etc/netplan/50-cloud-init.yaml` contoh:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.8.9/24]
      gateway4: 192.168.8.1
      nameservers:
        addresses: [8.8.8.8,1.1.1.1]
```

Lalu jalankan:

```bash
sudo netplan apply
```

---

## Contoh masalah & solusinya (kasus nyata dari tutorial ini)

* Error `paths.socketVMNet is a symlink` → copy binary ke `/opt/socket_vmnet/bin/`.
* Error `can't read "/private/etc/sudoers.d/lima"` → jalankan `limactl sudoers` dan install file sudoers di `/private/etc/sudoers.d/`.
* Host tidak bisa ping VM padahal VM bisa ping host → pastikan bridged benar atau gunakan route/portforward.

---
