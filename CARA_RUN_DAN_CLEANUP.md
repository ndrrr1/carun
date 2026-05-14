# Cara Menjalankan Program

Seluruh langkah dijalankan dari folder utama repository, yaitu folder yang berisi:

```text
soal_1  soal_2  soal_3
```

File tambahan yang perlu disiapkan di folder `~/Downloads`:

```text
amba_files.zip
notes.csv.enc
server
```

Masuk ke folder hasil extract repository. Contoh jika hasil extract ada di Downloads:

```bash
cd ~/Downloads/SISOP-4-2026-IT-124-main
```

Jika nama folder berbeda, sesuaikan dengan lokasi hasil extract. Cek posisi folder:

```bash
ls
```

Output yang diharapkan:

```text
soal_1  soal_2  soal_3
```

---

## Requirement Awal

### A. Update package list

```bash
sudo apt update
```

### B. Install dependency FUSE dan compiler

```bash
sudo apt install -y gcc pkg-config libfuse3-dev fuse3
```

### C. Install dependency Docker, Samba, dan tools tambahan

```bash
sudo apt install -y docker.io smbclient cifs-utils tree zip unzip curl xxd
```

### D. Aktifkan Docker dan FUSE

```bash
sudo systemctl enable --now docker
```

```bash
sudo modprobe fuse
```

### E. Aktifkan konfigurasi `allow_other`

```bash
sudo sed -i 's/^#user_allow_other/user_allow_other/' /etc/fuse.conf
```

```bash
grep -q '^user_allow_other' /etc/fuse.conf || echo 'user_allow_other' | sudo tee -a /etc/fuse.conf
```

---

# Cara Run Soal 1

Soal 1 berfokus pada FUSE yang membaca file perjalanan Mas Amba dari `amba_files.zip`, lalu membuat file virtual bernama `tujuan.txt`.

## A. Menyiapkan file ZIP bahan

Masuk ke folder Soal 1:

```bash
cd soal_1
```

Lepaskan mount lama jika masih ada:

```bash
fusermount3 -u mnt 2>/dev/null || true
```

Bersihkan runtime lama:

```bash
rm -rf amba_files
```

```bash
rm -rf mnt
```

```bash
rm -f kenz_rescue
```

```bash
rm -f fuse.log
```

Copy file bahan dari `Downloads` ke folder Soal 1:

```bash
cp ~/Downloads/amba_files.zip .
```

Buat mount point:

```bash
mkdir -p mnt
```

Cek file bahan:

```bash
ls
```

Output minimal yang diharapkan:

```text
amba_files.zip  kenz_rescue.c  mnt
```

Unzip file bahan:

```bash
unzip amba_files.zip
```

Cek isi folder bahan:

```bash
ls amba_files
```

Output yang diharapkan:

```text
1.txt  2.txt  3.txt  4.txt  5.txt  6.txt  7.txt
```

Cek salah satu isi file:

```bash
cat amba_files/1.txt
```

File tersebut harus memiliki baris koordinat:

```text
KOORD: -7.957
```

## B. Compile program Soal 1

```bash
gcc kenz_rescue.c $(pkg-config fuse3 --cflags --libs) -o kenz_rescue
```

## C. Jalankan FUSE Soal 1

```bash
./kenz_rescue amba_files mnt -f > fuse.log 2>&1 &
```

Tunggu proses mount:

```bash
sleep 2
```

## D. Cek hasil Soal 1

Cek isi mount point:

```bash
ls mnt
```

Output yang diharapkan:

```text
1.txt  2.txt  3.txt  4.txt  5.txt  6.txt  7.txt  tujuan.txt
```

Cek isi file virtual `tujuan.txt`:

```bash
cat mnt/tujuan.txt
```

Output yang diharapkan:

```text
Tujuan Mas Amba: -7.957382728443728,112.4698688227961,23:59 WIB
```

Pastikan `tujuan.txt` tidak ada di folder asli:

```bash
ls amba_files
```

Output yang diharapkan:

```text
1.txt  2.txt  3.txt  4.txt  5.txt  6.txt  7.txt
```

Stop FUSE Soal 1:

```bash
fusermount3 -u mnt
```

Kembali ke root repository:

```bash
cd ..
```

---

# Cara Run Soal 2

Soal 2 berfokus pada FUSE terenkripsi, file `.enc`, Docker server, dan client.

## A. Menyiapkan encrypted storage, mount point, dan server

Masuk ke folder Soal 2:

```bash
cd soal_2
```

Lepaskan mount lama jika masih ada:

```bash
fusermount3 -u fuse_mount 2>/dev/null || true
```

Hapus container lama jika ada:

```bash
sudo docker rm -f db_app 2>/dev/null || true
```

Bersihkan runtime lama:

```bash
rm -f fuse
```

```bash
rm -f client
```

```bash
rm -f fuse.log
```

```bash
rm -rf encrypted_storage/*
```

```bash
rm -rf fuse_mount/*
```

Buat folder backend untuk file terenkripsi:

```bash
mkdir -p encrypted_storage/tests
```

Buat folder mount point:

```bash
mkdir -p fuse_mount
```

Copy file `notes.csv.enc` dari `Downloads` ke backend:

```bash
cp ~/Downloads/notes.csv.enc encrypted_storage/tests/notes.csv.enc
```

Copy file `server` dari `Downloads` ke folder Soal 2:

```bash
cp ~/Downloads/server ./server
```

Beri permission executable pada `server`:

```bash
chmod +x server
```

Cek file penting:

```bash
ls encrypted_storage/tests
```

Output yang diharapkan:

```text
notes.csv.enc
```

Cek file server:

```bash
ls -l server
```

Output harus menunjukkan file `server` sudah ada dan executable.

## B. Compile program FUSE dan client

Compile FUSE:

```bash
gcc fuse.c $(pkg-config fuse3 --cflags --libs) -o fuse
```

Compile client:

```bash
gcc client.c -o client
```

Cek hasil compile:

```bash
ls
```

Output minimal yang diharapkan terdapat:

```text
client  fuse  server
```

## C. Jalankan FUSE dan cek enkripsi/dekripsi

Jalankan FUSE:

```bash
./fuse encrypted_storage fuse_mount -o allow_other -f > fuse.log 2>&1 &
```

Tunggu proses mount:

```bash
sleep 2
```

Cek file yang muncul di mount point:

```bash
ls fuse_mount/tests
```

Output yang diharapkan:

```text
notes.csv
```

Cek isi file hasil decrypt:

```bash
cat fuse_mount/tests/notes.csv
```

Output yang diharapkan:

```text
author,notes
admin,TEST_SUCCESS
```

Buat file uji baru lewat mount point:

```bash
echo "halo database" > fuse_mount/test.txt
```

Baca file uji:

```bash
cat fuse_mount/test.txt
```

Output yang diharapkan:

```text
halo database
```

Cek file asli di backend:

```bash
ls encrypted_storage
```

Output minimal yang diharapkan:

```text
test.txt.enc  tests
```

Cek isi file terenkripsi:

```bash
xxd encrypted_storage/test.txt.enc
```

Isi file seharusnya tidak terbaca sebagai teks normal.

## D. Jalankan Docker server dan client

Build Docker image:

```bash
sudo docker build -t soal-2-modul-4-sisop .
```

Jalankan container server:

```bash
sudo docker run -d --name db_app -p 9000:9000 -v "$(pwd)/fuse_mount:/app/db" soal-2-modul-4-sisop
```

Cek container:

```bash
sudo docker ps
```

Output yang diharapkan terdapat container:

```text
db_app
```

Cek log server:

```bash
sudo docker logs db_app
```

Jalankan client:

```bash
./client 127.0.0.1 9000
```

Output awal yang diharapkan:

```text
Connected to DB Server on 127.0.0.1:9000
Type EXIT to quit.
db>
```

Di dalam client, coba command:

```text
HELP
```

Untuk keluar dari client:

```text
EXIT
```

Stop container Soal 2:

```bash
sudo docker rm -f db_app
```

Stop FUSE Soal 2:

```bash
fusermount3 -u fuse_mount
```

Kembali ke root repository:

```bash
cd ..
```

---

# Cara Run Soal 3

Soal 3 berfokus pada Samba server, user/group, permission share, persistence data, dan logging.

## A. Menyiapkan folder volume dan permission

Masuk ke folder Soal 3:

```bash
cd soal_3
```

Buat folder `docs`:

```bash
mkdir -p data/docs
```

Buat folder `ebooks`:

```bash
mkdir -p data/ebooks
```

Buat folder `papers`:

```bash
mkdir -p data/papers
```

Buat folder `sourcecode`:

```bash
mkdir -p data/sourcecode
```

Buat folder log:

```bash
mkdir -p logs
```

Buat file log utama:

```bash
touch logs/libraryit.log
```

Beri permission executable pada `entrypoint.sh`:

```bash
chmod +x entrypoint.sh
```

Matikan Samba host agar port 445 tidak bentrok:

```bash
sudo systemctl stop smbd nmbd 2>/dev/null || true
```

## B. Build dan jalankan Docker Compose

Hapus container lama jika ada:

```bash
sudo docker rm -f libraryit-server 2>/dev/null || true
```

```bash
sudo docker rm -f libraryit-logger 2>/dev/null || true
```

Matikan compose lama jika ada:

```bash
sudo docker compose down --remove-orphans 2>/dev/null || true
```

Build image:

```bash
sudo docker compose build --no-cache
```

Jalankan container:

```bash
sudo docker compose up -d
```

Tunggu container siap:

```bash
sleep 5
```

Cek container:

```bash
sudo docker ps
```

Output yang diharapkan terdapat:

```text
libraryit-server
libraryit-logger
```

## C. Tes akses user dan permission share

Tes daftar share sebagai `member`:

```bash
smbclient -L //127.0.0.1 -U member --password='member123' -m SMB3
```

Output yang diharapkan:

```text
docs
ebooks
papers
IPC$
```

Share `sourcecode` tidak muncul untuk `member`.

Tes anonymous access:

```bash
smbclient -L //127.0.0.1 -N -m SMB3
```

Hasil yang diharapkan: anonymous access gagal.

Tes member membaca `docs`:

```bash
smbclient //127.0.0.1/docs -U member --password='member123' -m SMB3 -c "ls"
```

Buat file testing untuk `ebooks`:

```bash
echo "test ebook" > ~/test_ebook.txt
```

Upload file ke `ebooks` sebagai contributor:

```bash
smbclient //127.0.0.1/ebooks -U contributor --password='contrib456' -m SMB3 -c "put $HOME/test_ebook.txt test_ebook.txt; ls"
```

Buat file testing untuk `papers`:

```bash
echo "test paper" > ~/test_paper.txt
```

Upload file ke `papers` sebagai contributor:

```bash
smbclient //127.0.0.1/papers -U contributor --password='contrib456' -m SMB3 -c "put $HOME/test_paper.txt test_paper.txt; ls"
```

Buat file testing untuk `docs`:

```bash
echo "dokumen librarian" > ~/test_docs.txt
```

Upload file ke `docs` sebagai librarian:

```bash
smbclient //127.0.0.1/docs -U librarian --password='lib789' -m SMB3 -c "put $HOME/test_docs.txt test_docs.txt; ls"
```

Buat file testing contributor untuk `docs`:

```bash
echo "dokumen contributor" > ~/contributor_docs.txt
```

Tes contributor tidak bisa menulis ke `docs`:

```bash
smbclient //127.0.0.1/docs -U contributor --password='contrib456' -m SMB3 -c "put $HOME/contributor_docs.txt contributor_docs.txt; ls"
```

Output yang diharapkan:

```text
NT_STATUS_ACCESS_DENIED
```

atau upload gagal.

Tes member tidak bisa akses `sourcecode`:

```bash
smbclient //127.0.0.1/sourcecode -U member --password='member123' -m SMB3 -c "ls"
```

Output yang diharapkan:

```text
NT_STATUS_ACCESS_DENIED
```

Buat file testing untuk `sourcecode`:

```bash
echo "print('hello sourcecode')" > ~/main.py
```

Upload file ke `sourcecode` sebagai contributor:

```bash
smbclient //127.0.0.1/sourcecode -U contributor --password='contrib456' -m SMB3 -c "put $HOME/main.py main.py; ls"
```

## D. Cek persistence dan logging

Cek persistence data:

```bash
sudo find data -type f
```

Output yang diharapkan setelah testing:

```text
data/ebooks/test_ebook.txt
data/papers/test_paper.txt
data/docs/test_docs.txt
data/sourcecode/main.py
```

Artinya data berhasil tersimpan di folder host `data`, bukan hanya di dalam container.

Cek log final:

```bash
cat logs/libraryit.log | tail -30
```

Cek log container logger:

```bash
sudo docker logs libraryit-logger --tail=30
```

Format log yang diharapkan:

```text
[YYYY-MM-DD HH:MM:SS] [INFO] [username] [AKSI] [file/share]
[YYYY-MM-DD HH:MM:SS] [WARNING] [username] [DENIED] [file/share]
```

Stop Soal 3:

```bash
sudo docker compose down
```

Kembali ke root repository:

```bash
cd ..
```

---

# Bersih-Bersih Semua Hasil Run

Command ini dijalankan dari folder utama repository, yaitu folder yang berisi `soal_1`, `soal_2`, dan `soal_3`.

```bash
bash -lc 'set +e; fusermount3 -u soal_1/mnt 2>/dev/null || true; fusermount3 -u soal_2/fuse_mount 2>/dev/null || true; sudo docker rm -f db_app libraryit-server libraryit-logger 2>/dev/null || true; (cd soal_3 && sudo docker compose down --remove-orphans 2>/dev/null || sudo docker-compose down --remove-orphans 2>/dev/null || true); rm -rf soal_1/amba_files soal_1/mnt soal_1/kenz_rescue soal_1/fuse.log soal_1/amba_files.zip; rm -f soal_2/fuse soal_2/client soal_2/fuse.log; sudo rm -rf soal_2/encrypted_storage/* soal_2/fuse_mount/*; sudo rm -rf soal_3/data/docs/* soal_3/data/ebooks/* soal_3/data/papers/* soal_3/data/sourcecode/* soal_3/logs/*; mkdir -p soal_2/encrypted_storage soal_2/fuse_mount soal_3/data/docs soal_3/data/ebooks soal_3/data/papers soal_3/data/sourcecode soal_3/logs; touch soal_3/logs/libraryit.log; sudo chown -R "$USER:$USER" soal_3/data soal_3/logs soal_2/encrypted_storage soal_2/fuse_mount 2>/dev/null || true; chmod 755 soal_2/encrypted_storage soal_2/fuse_mount soal_3/data soal_3/data/docs soal_3/data/ebooks soal_3/data/papers soal_3/data/sourcecode soal_3/logs; chmod 644 soal_3/logs/libraryit.log; echo "Cleanup selesai. Repository sudah bersih dari hasil runtime."'
```

Setelah cleanup, cek struktur folder:

```bash
tree
```

Catatan: command cleanup ini hanya membersihkan hasil run di folder repository. File berikut tetap aman di `Downloads`:

```text
~/Downloads/amba_files.zip
~/Downloads/notes.csv.enc
~/Downloads/server
```
