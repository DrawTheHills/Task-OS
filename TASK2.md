## TASK 3 - LawakFS++ - A Cursed Filesystem with Censorship and Strict Access Policies

- Implementasi Filesystem FUSE dengan fungsi minimal:
    getattr, readdir, open, read, access
    Read-only: semua operasi tulis harus ditolak (EROFS)
    Blokir eksplisit syscall tulis: write, truncate, create, unlink, mkdir, rmdir, rename

- Fitur-fitur yang Harus Diimplementasikan:
    - Hidden Extension:
        - Semua file di-mount tanpa ekstensi saat ls
        - Akses file harus tetap mengarah ke file asli dengan ekstensi
    - Time-Based Access untuk File "Secret"
        - File dengan nama dasar secret hanya bisa diakses antara jam tertentu (default: 08:00–18:00)
        - Di luar jam: kembalikan ENOENT pada operasi access, getattr, readdir, dll.
    - Dynamic Content Filtering:
        - Teks: kata-kata tertentu (dari konfigurasi) diganti dengan "lawak" (case-insensitive)
        - Biner: isi ditampilkan sebagai string base64 saat dibaca
    - Logging:
        - Log semua operasi read dan access yang berhasil ke /var/log/lawakfs.log
        - Format log: [YYYY-MM-DD HH:MM:SS] [UID] [ACTION] [PATH]
    - Konfigurasi Eksternal (lawak.conf):
        - SECRET_FILE_BASENAME
        - ACCESS_START dan ACCESS_END
        - FILTER_WORDS (kata-kata yang diganti jadi “lawak”)

### Code
**Lawakfs.c**
```c
#define FUSE_USE_VERSION 31
#include <fuse3/fuse.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <dirent.h>
#include <stdlib.h>
#include <limits.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <time.h>
#include <pwd.h>

char source_dir[PATH_MAX];
char secret_name[NAME_MAX] = "secret";
int access_start_hour = 8;
int access_end_hour = 18;

void log_action(const char *action, const char *path) {
    fprintf(stderr, "[DEBUG] log_action called for %s %s\n", action, path);

    FILE *log_fp = fopen("lawakfs.log", "a");
    if (!log_fp) {
        perror("[ERROR] fopen log");
        return;
    }

    time_t t = time(NULL);
    struct tm *lt = localtime(&t);
    uid_t uid = getuid();

    fprintf(log_fp, "[%04d-%02d-%02d %02d:%02d:%02d] [%d] [%s] [%s]\n",
            lt->tm_year + 1900, lt->tm_mon + 1, lt->tm_mday,
            lt->tm_hour, lt->tm_min, lt->tm_sec,
            uid, action, path);

    fclose(log_fp);
    fprintf(stderr, "[DEBUG] log_action write successful\n");
}

void parse_config() {
    FILE *fp = fopen("lawak.conf", "r");
    if (!fp) {
        perror("Failed to open config");
        return;
    }

    char line[256];
    while (fgets(line, sizeof(line), fp)) {
        if (strncmp(line, "SECRET_FILE_BASENAME=", 22) == 0) {
            sscanf(line + 22, "%s", secret_name);
        } else if (strncmp(line, "ACCESS_START=", 13) == 0) {
            sscanf(line + 13, "%d", &access_start_hour);
        } else if (strncmp(line, "ACCESS_END=", 11) == 0) {
            sscanf(line + 11, "%d", &access_end_hour);
        }
    }
    fclose(fp);
}

int is_secret_file(const char *name) {
    return strncmp(name, secret_name, strlen(secret_name)) == 0;
}

int check_secret_access() {
    time_t t = time(NULL);
    struct tm *tm = localtime(&t);
    int hour = tm->tm_hour;

    return hour >= access_start_hour && hour < access_end_hour;
}

static char *find_real_filename(const char *dirname, const char *name_wo_ext) {
    static char result[PATH_MAX];
    DIR *dp = opendir(dirname);
    struct dirent *de;

    if (!dp) return NULL;

    while ((de = readdir(dp)) != NULL) {
        char *dot = strrchr(de->d_name, '.');
        if (dot) {
            size_t len = dot - de->d_name;
            if (strncmp(de->d_name, name_wo_ext, len) == 0 &&
                strlen(name_wo_ext) == len) {
                snprintf(result, PATH_MAX, "%s/%s", dirname, de->d_name);
                closedir(dp);
                return result;
            }
        }
    }

    closedir(dp);
    return NULL;
}

static int lawakfs_getattr(const char *path, struct stat *stbuf,
                           struct fuse_file_info *fi) {
    (void) fi;
    int res;
    char real_path[PATH_MAX];

    if (strcmp(path, "/") == 0) {
        snprintf(real_path, PATH_MAX, "%s", source_dir);
    } else {
        const char *name = path + 1;
        if (is_secret_file(name) && !check_secret_access()) {
            return -ENOENT;
        }
        char *full_real = find_real_filename(source_dir, name);
        if (!full_real) return -ENOENT;
        snprintf(real_path, PATH_MAX, "%s", full_real);
    }

    res = lstat(real_path, stbuf);
    if (res == -1)
        return -errno;

    return 0;
}

static int lawakfs_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                           off_t offset, struct fuse_file_info *fi,
                           enum fuse_readdir_flags flags) {
    (void) offset;
    (void) fi;
    (void) flags;

    DIR *dp;
    struct dirent *de;
    char full_path[PATH_MAX];

    snprintf(full_path, PATH_MAX, "%s%s", source_dir, path);
    dp = opendir(full_path);
    if (dp == NULL)
        return -errno;

    filler(buf, ".", NULL, 0, 0);
    filler(buf, "..", NULL, 0, 0);

    while ((de = readdir(dp)) != NULL) {
        if (de->d_type == DT_REG) {
            char *dot = strrchr(de->d_name, '.');
            if (dot) {
                size_t len = dot - de->d_name;
                char name_wo_ext[NAME_MAX];
                strncpy(name_wo_ext, de->d_name, len);
                name_wo_ext[len] = '\0';
                if (is_secret_file(name_wo_ext) && !check_secret_access()) continue;
                filler(buf, name_wo_ext, NULL, 0, 0);
            } else {
                if (is_secret_file(de->d_name) && !check_secret_access()) continue;
                filler(buf, de->d_name, NULL, 0, 0);
            }
        } else {
            filler(buf, de->d_name, NULL, 0, 0);
        }
    }

    closedir(dp);
    return 0;
}

static int lawakfs_open(const char *path, struct fuse_file_info *fi) {
    char real_path[PATH_MAX];
    const char *name = path + 1;
    if (is_secret_file(name) && !check_secret_access()) {
        return -ENOENT;
    }

    char *full_real = find_real_filename(source_dir, name);
    if (!full_real) return -ENOENT;

    snprintf(real_path, PATH_MAX, "%s", full_real);

    int fd = open(real_path, O_RDONLY);
    if (fd == -1)
        return -errno;

    close(fd);
    return 0;
}

static int lawakfs_read(const char *path, char *buf, size_t size, off_t offset,
                        struct fuse_file_info *fi) {
    (void) fi;
    char real_path[PATH_MAX];
    const char *name = path + 1;
    if (is_secret_file(name) && !check_secret_access()) {
        return -ENOENT;
    }

    char *full_real = find_real_filename(source_dir, name);
    if (!full_real) return -ENOENT;
    snprintf(real_path, PATH_MAX, "%s", full_real);

    int fd = open(real_path, O_RDONLY);
    if (fd == -1)
        return -errno;

    int res = pread(fd, buf, size, offset);
    if (res == -1)
        res = -errno;

    close(fd);
    if (res > 0) {
        log_action("READ", path);
    }

    return res;
}

static int lawakfs_access(const char *path, int mask) {
    char real_path[PATH_MAX];

    if (strcmp(path, "/") == 0) {
        snprintf(real_path, PATH_MAX, "%s", source_dir);
    } else {
        const char *name = path + 1;
        if (is_secret_file(name) && !check_secret_access()) {
            return -ENOENT;
        }

        char *full_real = find_real_filename(source_dir, name);
        if (!full_real) return -ENOENT;

        snprintf(real_path, PATH_MAX, "%s", full_real);
    }

    if (access(real_path, mask) == -1)
        return -errno;

    log_action("ACCESS", path);
    return 0;
}

static struct fuse_operations lawakfs_oper = {
    .getattr = lawakfs_getattr,
    .readdir = lawakfs_readdir,
    .open    = lawakfs_open,
    .read    = lawakfs_read,
    .access  = lawakfs_access,
};

int main(int argc, char *argv[]) {
    if (argc < 3) {
        fprintf(stderr, "Usage: %s <source_dir> <mount_point>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    realpath(argv[1], source_dir);
    printf("Using source_dir: %s\n", source_dir);

    parse_config();

    argv[1] = argv[2];
    argc--;

    return fuse_main(argc, argv, &lawakfs_oper, NULL);
}
```

```c
SECRET_FILE_BASENAME=secret
ACCESS_START=08:00
ACCESS_END=18:00
```

### Explanation

**Lawakfs.c**

- **Read-only Filesystem**
    Semua file hanya bisa dibaca, dan tidak ada operasi tulis yang diizinkan. Fungsi seperti `write`, `create`, `rm`, dan lainnya tidak diimplementasikan untuk menjaga filesystem tetap read-only.

- **Parsing Konfigurasi (parse_config)**
    Fungsi ini membaca file konfigurasi eksternal lawak.conf, yang memuat:
    - SECRET_FILE_BASENAME: nama dasar file yang dianggap sebagai file rahasia.
    - ACCESS_START dan ACCESS_END: waktu di mana file rahasia boleh diakses.
    Nilai-nilai ini disimpan dalam variabel global untuk digunakan oleh seluruh program.

- **Kontrol Akses Berdasarkan Waktu (check_secret_access)**
    Fungsi ini membatasi akses ke file "secret" hanya saat jam kerja, yaitu antara ACCESS_START dan ACCESS_END. Jika diakses di luar jam tersebut, file dianggap tidak ada (ENOENT).

- **Menyembunyikan Ekstensi File (readdir)**
    Dalam fungsi readdir, saat filesystem membaca isi direktori:
    - File regular dengan ekstensi akan ditampilkan tanpa ekstensinya (contoh: file.txt → tampil sebagai file).
    - Nama file ini diproses ulang untuk pemetaan ke file asli saat open, read, dan getattr.

- **Mencari Nama File Asli (find_real_filename)**
    Fungsi ini mencari file asli di direktori sumber berdasarkan nama tanpa ekstensi (yang ditampilkan ke pengguna), dan mengembalikan path lengkap file dengan ekstensi aslinya.

- **Pemrosesan Akses File (getattr, open, read, access)**
    - Fungsi-fungsi tersebut bertugas menerjemahkan path dari user ke path asli di direktori sumber.
    - Jika file adalah "secret" dan akses dilakukan di luar jam, fungsi langsung mengembalikan ENOENT.

- **Log**
    - Membuka file lawakfs.log dengan mode append ("a").
    - Menulis data log sesuai format.
    - Menutup file log kembali.

- **Main**
    - Memastikan bahwa argumen input adalah source_dir dan mount_point.
    - Menyimpan path absolut direktori sumber.
    - Memanggil parse_config() untuk memuat konfigurasi dari lawak.conf.


**Lawakfs.conf**

- **SECRET_FILE_BASENAME=secret**
    Menentukan nama dasar file yang dianggap sebagai file rahasia.

- **ACCESS_START=08:00**
    Menentukan jam mulai kapan file rahasia boleh diakses.

- **ACCESS_END=18:00**
    Menentukan jam akhir kapan file rahasia masih boleh diakses.

### Screnshoot

![Screenshot 2025-06-22 181935](https://github.com/user-attachments/assets/16f3a986-dd21-4659-86cf-520b2e5080e6)

![Screenshot 2025-06-22 183111](https://github.com/user-attachments/assets/afe0a055-040f-4969-bc80-5cc2080a1c34)

