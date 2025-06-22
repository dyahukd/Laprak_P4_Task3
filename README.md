
# Laprak\_P4\_TrollFS - "Drama Troll Filesystem"

## Soal Praktikum 4 - Sistem Operasi

### **Judul**: Drama Troll

### **Deskripsi:**

Dalam soal ini, kita diminta membuat sebuah filesystem jebakan menggunakan FUSE. Filesystem ini akan bereaksi tergantung siapa user yang mengakses dan apakah "jebakan" sudah aktif atau belum. Fokus dari proyek ini adalah membuat filesystem dinamis dengan perlakuan berbeda untuk user tertentu (dalam hal ini: DainTontas) dan juga efek permanen (persistent trap) setelah trigger dilakukan.

---

## Soal A - Pembuatan User

### **Perintah Terminal**:

```bash
sudo useradd -m DainTontas
sudo passwd DainTontas

sudo useradd -m SunnyBolt
sudo passwd SunnyBolt

sudo useradd -m Ryeku
sudo passwd Ryeku
```

### **Penjelasan:**

Perintah `useradd -m` akan membuat user baru sekaligus home directory-nya. Sementara `passwd` digunakan untuk mengatur password masing-masing user. Ketiga user ini merepresentasikan karakter utama di cerita trolling yaitu DainTontas (target) serta SunnyBolt dan Ryeku (perancang jebakan).

---

## Soal B & C - Filesystem Jebakan & Perilaku Berdasarkan User

### **Struktur File FUSE:**

- `/very_spicy_info.txt` → file yang akan memunculkan konten berbeda tergantung siapa yang membaca.
- `/upload.txt` → file kosong yang menjadi trigger untuk menjebak DainTontas.

### **Kode Lengkap (Revisi Trap):**

```c
#define FUSE_USE_VERSION 28
#include <fuse.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>
#include <fcntl.h>
#include <dirent.h>
#include <pwd.h>
#include <fuse/fuse_common.h>
#include <sys/types.h>

const char *ascii_art =
" _____    _ _    __              _ _                       _                                        _ \n"
"|  ___|__| | |  / _| ___  _ __  (_) |_    __ _  __ _  __ _(_)_ __    _ __ _____      ____ _ _ __ __| |\n"
"| |_ / _ \ | | | |_ / _ \| '__| | | __|  / _` |/ _` |/ _` | | '_ \  | '__/ _ \\ \ /\ / / _` | '__/ _` |\n"
"|  _|  __/ | | |  _| (_) | |    | | |_  | (_| | (_| | (_| | | | | | | | |  __/\ V  V / (_| | | | (_| |\n"
"|_|  \___|_|_| |_|  \___/|_|    |_|\__|  \__,_|\__, |\__,_|_|_| |_| |_|  \___| \_/\_/ \__,_|_|  \__,_|\n"
"                                               |___/                                                  \n";

const char *trigger_path = "/tmp/.troll_triggered";
static const char *file1 = "/very_spicy_info.txt";
static const char *file2 = "/upload.txt";

static int troll_getattr(const char *path, struct stat *stbuf) {
    memset(stbuf, 0, sizeof(struct stat));
    if (strcmp(path, "/") == 0) {
        stbuf->st_mode = S_IFDIR | 0755;
        stbuf->st_nlink = 2;
    } else if (strcmp(path, file1) == 0 || strcmp(path, file2) == 0) {
        stbuf->st_mode = S_IFREG | 0644;
        stbuf->st_nlink = 1;
        stbuf->st_size = 1024;
    } else {
        return -ENOENT;
    }
    return 0;
}

static int troll_readdir(const char *path, void *buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info *fi) {
    if (strcmp(path, "/") != 0) return -ENOENT;
    filler(buf, ".", NULL, 0);
    filler(buf, "..", NULL, 0);
    filler(buf, file1 + 1, NULL, 0);
    filler(buf, file2 + 1, NULL, 0);
    return 0;
}

static int troll_open(const char *path, struct fuse_file_info *fi) {
    if (strcmp(path, file1) != 0 && strcmp(path, file2) != 0) return -ENOENT;
    return 0;
}

static int troll_read(const char *path, char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    const char *message = "Default file content.";
    int trap_on = access(trigger_path, F_OK) == 0;
    const char *username = getpwuid(fuse_get_context()->uid)->pw_name;

    if (strstr(path, ".txt") != NULL) {
        if (trap_on) {
            message = ascii_art;
        } else if (strcmp(path, file1) == 0 && strcmp(username, "DainTontas") == 0) {
            message = "Very spicy internal developer information: leaked roadmap.docx";
        } else {
            message = "DainTontas' personal secret!!.txt";
        }
    } else if (strcmp(path, file2) == 0) {
        message = "";
    }

    size_t len = strlen(message);
    if (offset < len) {
        if (offset + size > len) size = len - offset;
        memcpy(buf, message + offset, size);
    } else {
        size = 0;
    }

    return size;
}

static int troll_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi) {
    if (strcmp(path, file2) == 0) {
        const char *username = getpwuid(fuse_get_context()->uid)->pw_name;
        if (strcmp(username, "DainTontas") == 0) {
            FILE *f = fopen(trigger_path, "w");
            if (f != NULL) {
                fputs("TRIGGERED", f);
                fclose(f);
            }
        }
        return size;
    }
    return -EACCES;
}

static struct fuse_operations troll_oper = {
    .getattr = troll_getattr,
    .readdir = troll_readdir,
    .open    = troll_open,
    .read    = troll_read,
    .write   = troll_write,
};

int main(int argc, char *argv[]) {
    umask(0);
    return fuse_main(argc, argv, &troll_oper, NULL);
}
```

---

### **Penjelasan Kode (versi mahasiswa):**

- `getattr()`: Nentuin atribut dari file & folder virtual di FUSE.
- `readdir()`: Buat nampilin isi direktori `/`, yaitu 2 file jebakan.
- `open()`: Cek apakah file yang dibuka valid.
- `read()`: Di sinilah magic-nya.
  - Kalau belum dijebak (trigger off):
    - DainTontas buka spicy → lihat roadmap palsu.
    - User lain → lihat aib DainTontas.
  - Kalau trigger udah ON (udah upload):
    - Semua file `.txt` langsung munculin ASCII "Fell for it again reward".
- `write()`: Kalau user DainTontas nulis ke `upload.txt`, file trigger dibuat di `/tmp/.troll_triggered`, jadi efeknya permanen.
- `umask(0)`: Supaya file punya permission full sesuai yang ditetapkan.

---

## Soal D - ASCII Trap Trigger & Persistence

### **Trigger di Terminal:**

```bash
sudo ./trollfs -o allow_other /mnt/troll

# Lalu dari user DainTontas:
sudo -u DainTontas bash
cd /mnt/troll
echo upload > upload.txt  # TRIGGER!!
cat very_spicy_info.txt   # ASCII keluar
```

### **Hasil Akhir:**

Setelah `upload.txt` ditulis oleh DainTontas:

- Semua file `.txt` (bukan cuma spicy) akan nampilin ASCII art.
- Efeknya tetap aktif meski FUSE dimatikan, karena trigger berupa file `/tmp/.troll_triggered` di luar FUSE (persistent).

### **Kendala:**

- Pertama kali nyoba echo upload > upload.txt bisa muncul, tapi selama masa revisi semuanya error dikarenakan lupa nge-run apa saja. Jadinya mulai dari awal, dan stuck di soal d waktu echo upload > upload.txt, keluarnya permission denied, function not implemented, atau no such file or directory, udah coba benerin tapi hasilnya sama aja, kalaupun bisa, hasil cat very_spicy_info.txt nya kalimat "upload".

---

---

## 🖼 Screenshot

1. `ls /mnt/troll` → ada dua file
2. `cat very_spicy_info.txt` sebelum dan sesudah trigger
3. Bukti `/tmp/.troll_triggered` terbentuk

---
