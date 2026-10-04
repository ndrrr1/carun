# Demo Singkat 11–20 — K12

Jalankan sebagai root di console node GNS3 yang disebutkan, setelah konfigurasi 1–20 selesai. Skrip `/root/*.sh` dan `/root/dns-edit.py` berasal dari README konfigurasi.

## 11. Reverse proxy dan pembagian request

**Soal:** WWW lewat Penny dibagi ke Obladi/Desmond; static lewat Abbey dibagi ke Oblada/Molly.

**Console Alpha:**
```bash
for host in www.k12.com static.k12.com; do
    printf '\n=== %s ===\n' "$host"
    for i in $(seq 1 10); do
        curl --noproxy '*' --max-time 10 -sS -D - -o /dev/null "http://$host/" \
            | tr -d '\r' | grep -Ei '^(HTTP/|X-Backend:)'
    done
done
```
**Ngapain:** Mengirim 10 request per domain. `HTTP` menunjukkan status; `X-Backend` menunjukkan server yang melayani. Harus muncul **200** dan **kedua backend** untuk setiap domain; tidak harus bergantian persis.

## 12. Login halaman admin

**Soal:** `/admin` hanya bisa diakses dengan username dan password yang benar.

**Console Alpha:**
```bash
echo 'TANPA LOGIN:'
curl --noproxy '*' --max-time 10 -sS -I http://www.k12.com/admin/
echo 'PASSWORD SALAH:'
curl --noproxy '*' --max-time 10 -sS -I -u 'prabs:salah' http://www.k12.com/admin/
echo 'LOGIN BENAR:'
curl --noproxy '*' --max-time 10 -sS -i -u 'prabs:pakar_pinter_jadi_gob***' http://www.k12.com/admin/
echo 'TANPA SLASH, TANPA LOGIN:'
curl --noproxy '*' --max-time 10 -sS -I http://www.k12.com/admin
```
**Ngapain:** Menguji akses tanpa login, password salah, login benar, dan `/admin` tanpa slash. `-u` mengirim kredensial; `-I` melihat header; `-i` menampilkan header dan isi. Hasil berurutan: **401, 401, 200, 401**. Login benar menampilkan halaman Admin Penny. Tiga `*` adalah bagian dari password.

## 13. Redirect ke domain utama

**Soal:** Akses melalui IP/nama Penny diarahkan ke WWW dengan 301; Abbey ke static dengan 302. Path dan query tetap dibawa.

**Console Alpha:**
```bash
for url in http://192.217.3.2/ http://penny.k12.com/ \
    http://192.217.2.2/ http://abbey.k12.com/ \
    'http://penny.k12.com/arsip/?cek=1' 'http://abbey.k12.com/profil?cek=1'; do
    printf '\n%s\n' "$url"
    curl --noproxy '*' --max-time 10 -sS -I "$url" \
        | tr -d '\r' | grep -Ei '^(HTTP/|Location:)'
done
```
**Ngapain:** Melihat kode redirect dan alamat tujuan pada `Location`. Penny harus **301 → www.k12.com**, Abbey **302 → static.k12.com**. Bagian `/arsip/?cek=1` dan `/profil?cek=1` harus tetap ada.

## 14. IP asli klien pada log

**Soal:** Backend mencatat IP asli klien dan hostname tujuan walaupun request melewati proxy.

**Console Alpha, lalu ulangi di Delta:**
```bash
hostname
node=$(hostname -s)
for i in $(seq 1 6); do
    curl --noproxy '*' --max-time 10 -sS -o /dev/null "http://www.k12.com/?demo=14-$node-$i"
    curl --noproxy '*' --max-time 10 -sS -o /dev/null "http://static.k12.com/profil?demo=14-$node-$i"
done
```
**Ngapain:** Membuat request dari dua klien. `hostname` menunjukkan node; `demo=14-...` memberi penanda pada log.

**Console Obladi, lalu ulangi di Desmond:**
```bash
hostname
grep 'demo=14-' /var/log/apache2/vault-access.log | tail -n 8
```
**Console Oblada, lalu ulangi di Molly:**
```bash
hostname
grep 'demo=14-' /var/log/nginx/core-access.log | tail -n 8
```
**Ngapain:** `grep` mencari request demo; `tail` mengambil delapan baris terakhir. IP klien harus **Alpha 192.217.4.2** dan **Delta 192.217.5.2**. IP koneksi proxy tercatat terpisah: Penny **192.217.3.2** atau Abbey **192.217.2.2**. Host harus WWW/static sesuai request.

## 15. Eternal PHP dan Orion statis

**Soal:** Penny menjalankan PHP pada `/eternal/`; Abbey melayani file statis pada `/orion/` dan menolak PHP di jalur itu.

**Console Alpha:**
```bash
curl --noproxy '*' --max-time 10 -sS -i http://www.k12.com/eternal/
curl --noproxy '*' --max-time 10 -sS -i http://www.k12.com/eternal/index.php
curl --noproxy '*' --max-time 10 -sS -i http://static.k12.com/orion/
curl --noproxy '*' --max-time 10 -sS -I http://static.k12.com/orion/contoh.php
```
**Ngapain:** Membuka dua alamat Eternal, halaman Orion, lalu mencoba file PHP pada Orion. Hasil: **200, 200, 200, 403**. Eternal harus menampilkan **hasil perhitungan PHP: 5**; Orion menampilkan halaman statisnya.

## 16. Benchmark web

**Soal:** Uji WWW dan static dengan 250 request, 10 request bersamaan, lalu bandingkan performanya.

**Console Alpha:**
```bash
mkdir -p /root/bukti-demo
ab -l -n 250 -c 10 http://www.k12.com/ 2>&1 | tee /root/bukti-demo/16-www.txt
ab -l -n 250 -c 10 http://static.k12.com/ 2>&1 | tee /root/bukti-demo/16-static.txt
```
**Ngapain:** `ab` menguji beban; `-n 250` total request; `-c 10` request bersamaan; `-l` mengabaikan perbedaan panjang body antarbackend sebagai penyebab kegagalan. `tee` menyimpan output. Periksa **Complete requests 250**, **Failed requests 0**, dan tidak ada respons non-2xx. Bandingkan **Requests per second** dan baris **Time per request** yang sama.

## 17. TXT record klien

**Soal:** TXT record Alpha–Epsilon berisi nama masing-masing klien dan tersedia pada kedua DNS.

**Console Alpha:**
```bash
for dns in 192.217.1.2 192.217.1.3; do
    echo "DNS $dns"
    for host in alpha beta gamma delta epsilon; do
        printf '%s: ' "$host"
        dig @"$dns" "$host.k12.com" TXT +short
    done
done
```
**Ngapain:** `dig @IP` bertanya langsung ke Prab/Tedd; `TXT` meminta record teks; `+short` menampilkan jawaban ringkas. Isinya harus **alpha, beta, gamma, delta, epsilon** sesuai nama yang ditanyakan.

## 18. Cache DNS dan TTL 15 detik

**Soal:** Buktikan cache menyimpan IP lama sampai TTL habis, kemudian mengambil IP baru. Gunakan dua console: Prab dan Alpha.

**A. Console Prab:**
```bash
python3 /root/dns-edit.py ttl15
dig @192.217.1.2 abbey.k12.com A +noall +answer
```
**Ngapain:** Menyiapkan IP Abbey **192.217.2.2** dengan TTL **15 detik**; `dig` menampilkan IP dan TTL.

**B. Console Alpha:**
```bash
if ! command -v dnsmasq >/dev/null 2>&1 || [ ! -f /root/cache18.conf ]; then
    bash /root/18-cache.sh setup
fi
bash /root/18-cache.sh reset
bash /root/18-cache.sh watch
```
**Ngapain:** Menyiapkan cache jika belum tersedia, mengosongkan cache awal, lalu membandingkan cache Alpha dan jawaban Prab selama sekitar 30 detik.

**C. Console Prab, sekitar detik ke-3 saat watch Alpha masih berjalan:**
```bash
python3 /root/dns-edit.py fake
```
**Ngapain:** Mengganti IP Abbey menjadi **203.0.113.77**. Jangan reset cache selama watch berjalan.

**D. Console Alpha, setelah watch selesai:**
```bash
cat /root/bukti/ttl-abbey.txt
dig @192.217.1.2 k12.com SOA +short
dig @192.217.1.3 k12.com SOA +short
dig @192.217.1.3 abbey.k12.com A +norecurse +noall +comments +answer
```
**Ngapain:** `cat` menampilkan rekaman; `SOA` membandingkan serial zona; query terakhir memeriksa IP baru pada Tedd. Rekaman harus menunjukkan **lama/lama → cache lama tetapi Prab baru → baru/baru**. TTL cache menurun sebelum IP berubah. Serial kedua DNS sama setelah sinkronisasi.

**E. Console Prab — pulihkan IP:**
```bash
python3 /root/dns-edit.py restore
```
**Ngapain:** Mengembalikan IP Abbey ke **192.217.2.2** dan TTL normal.

**F. Console Alpha:**
```bash
bash /root/18-cache.sh stop
dig @192.217.1.2 abbey.k12.com A +short
dig @192.217.1.3 abbey.k12.com A +short
curl --noproxy '*' --max-time 10 -sS -I http://static.k12.com/
```
**Ngapain:** Menghentikan cache uji dan memeriksa pemulihan. Kedua DNS harus menjawab **192.217.2.2**, web **200**. Jika Tedd belum berubah, tunggu sinkronisasi lalu ulangi query.

## 19. CNAME eksternal

**Soal:** `outbound.k12.com` menjadi alias `http.badssl.com`; periksa DNS dan respons webnya.

**Console Alpha:**
```bash
dig @192.217.1.2 outbound.k12.com CNAME +short
dig @192.217.1.3 outbound.k12.com CNAME +short
dig outbound.k12.com A +noall +answer
curl --noproxy '*' --max-time 20 -sS -i http://outbound.k12.com/
```
**Ngapain:** Memeriksa alias pada kedua DNS, resolusi IP, dan respons HTTP. CNAME harus **http.badssl.com.** CNAME tidak mengganti header HTTP `Host`, sehingga respons akses lewat alias bisa berbeda.

**Masih di Alpha — bandingkan dengan Host yang sama:**
```bash
mkdir -p /root/bukti-demo
curl --noproxy '*' --max-time 20 -fsS -H 'Host: http.badssl.com' -D - \
    -o /root/bukti-demo/19-alias.html http://outbound.k12.com/ &&
curl --noproxy '*' --max-time 20 -fsS -D - \
    -o /root/bukti-demo/19-target.html http://http.badssl.com/ &&
if cmp -s /root/bukti-demo/19-alias.html /root/bukti-demo/19-target.html; then
    echo 'BODY SAMA'
else
    echo 'BODY BERBEDA'
fi
```
**Ngapain:** `-H` menyamakan Host; `-D -` menampilkan header; `-o` menyimpan body; `cmp` membandingkan isi. `-f` dan `&&` memastikan perbandingan hanya berjalan setelah kedua unduhan berhasil. Catat hasil yang benar-benar muncul.

## 20. Autostart setelah Stop/Start

**Soal:** IP, routing, DNS, dan web kembali aktif otomatis setelah seluruh node di-Stop lalu di-Start.

**A. Sebelum Stop, console Prab — cek kondisi akhir DNS:**
```bash
dig @192.217.1.2 k12.com SOA +short
dig @192.217.1.3 k12.com SOA +short
dig @192.217.1.2 abbey.k12.com A +short
dig @192.217.1.3 abbey.k12.com A +short
```
**Ngapain:** Pastikan serial kedua DNS sama dan kedua IP Abbey **192.217.2.2**. Setelah cocok, jalankan blok berikut **di Prab, lalu di Tedd**:
```bash
bash /root/20-autostart.sh
```
**Ngapain:** Memperbarui snapshot konfigurasi dan cache paket untuk boot, agar perubahan DNS nomor 18–19 ikut tersimpan. Tunggu skrip selesai pada kedua node sebelum Stop.

**B. Di GNS3, pastikan Start command setiap node berisi:**
```text
/bin/bash -c '/bin/bash /root/boot.sh || tail -n 60 /root/lab-boot.log; exec /bin/bash'
```
**Ngapain:** Menjalankan pemulihan otomatis saat node dinyalakan; menampilkan log jika gagal; kemudian membuka console. Isi di kolom **Start command**, bukan dijalankan manual saat pembuktian.

**C. Console Alpha, sebelum Stop:**
```bash
mkdir -p /root/bukti-demo
date -Is | tee /root/bukti-demo/20-sebelum-stop.txt
```
**Ngapain:** Menyimpan tanggal dan waktu sebelum restart.

**D. Di GNS3:** Stop semua node, lalu Start per kelompok: **Rootkit → Prab/Tedd → Obladi/Desmond/Oblada/Molly → Penny/Abbey → Alpha/Beta/Gamma/Delta/Epsilon**. Tunggu kelompok sebelumnya selesai boot sebelum lanjut.

**E. Setelah Start, console semua 14 node, satu per satu:**
```bash
hostname
ip -br -4 address
ip -4 route
cat /etc/resolv.conf
tail -n 12 /root/lab-boot.log
```
**Ngapain:** Memeriksa nama node, IP, routing, resolver DNS, dan log boot. Harus ada **BOOT OK** dari boot terbaru. Jangan menyalakan layanan secara manual sebelum pengecekan.

**F. Console Alpha, setelah semua node siap:**
```bash
cat /root/bukti-demo/20-sebelum-stop.txt
date -Is
ping -c 3 -W 2 1.1.1.1
dig @192.217.1.2 k12.com SOA +short
dig @192.217.1.3 k12.com SOA +short
dig @192.217.1.2 abbey.k12.com A +short
dig @192.217.1.3 abbey.k12.com A +short
dig @192.217.1.3 -x 192.217.1.4 +short
dig alpha.k12.com TXT +short
dig outbound.k12.com CNAME +short
for url in http://www.k12.com/ http://static.k12.com/ \
    http://www.k12.com/arsip/ http://static.k12.com/profil \
    http://www.k12.com/eternal/ http://static.k12.com/orion/; do
    curl --noproxy '*' --max-time 10 -sS -o /dev/null -w "$url HTTP %{http_code}\n" "$url"
done
```
**Ngapain:** Membandingkan waktu sebelum/sesudah, menguji internet dengan tiga ping, memeriksa DNS termasuk reverse (`-x`), TXT, CNAME, dan enam alamat web. IP Abbey harus normal; keenam web harus **200**.

**G. Ulangi command nomor 11–15 dan 17 setelah restart**, pada console yang sama seperti sebelumnya. Ini membuktikan pembagian request, login, redirect, log IP klien, PHP/statis, dan seluruh TXT record tetap bekerja otomatis.
