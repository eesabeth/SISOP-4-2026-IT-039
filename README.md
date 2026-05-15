# SISOP-4-2026-IT-039
## Laporan Resmi Modul 4 Sisop oleh Elisabeth L. S. S. | 039
### Soal 1
### Penjelasan

Soal ini meminta kita untuk mengambil arsip `amba_files` lalu menerapkan sistem kerja Linux yaitu **Fuse** untuk mengerjakan beberapa perintah. Hal-hal yang diminta dari soal ini adalah:

a. Mengambil `amba_files` dari *Flashdisk* yang telah diberikan  
b. Membuat program `kenz_rescue.c` yang menerima argumen `<source_directory>` dan `<mount_directory>`  
c. Saat `kenz_rescue.c` di-*mount*, ketujuh file `1.txt` sampai `7.txt` harus muncul mount directory sama persis dengan source  
d. Setelah `./kenz_rescue.c amba_files mnt`, hasil `cat mnt/1.txt` sama dengan `cat amba_files/1.txt`  
e. Membuat file virtual `tujuan.txt`di mount directory. File harus muncul saat `ls mnt/`, ukurannya stabil saat di-*stat*, dan tidak memiliki file fisik di `amba_files`  
f. Saat `cat mnt/tujuan.txt`, menghasilkan output one liner dengan format **"Tujuan Mas Amba: <gabungan_fragmen>"**

#### Instalasi `fuse3`
```
sudo apt install libfuse3-dev fuse3
```

#### a. Mengambil `amba_files` dari *Flashdisk* yang telah diberikan
Menggunakan `gdown` untuk men-*download* `amba_files` dari link google drive.  
```
gdown "https://drive.google.com/file/d/1nLXFhptDo2mnUlZsw8pTWyAVpV49W20U/view?usp=drive_link"
```
*Unzip* `amba_files.zip`, lalu remove zip nya meninggalkan hanya `amba_files`.
```
unzip amba_files.zip
rm amba_files.zip
```
Docum:

Buat mount directory sebelum membuat `kenz_rescue.c`.
```
mkdir mnt
```
#### b. Membuat program `kenz_rescue.c` yang menerima argumen `<source_directory>` dan `<mount_directory>`  
```
#define FUSE_USE_VERSION 31

#include <fuse.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <stddef.h>
#include <stdlib.h>
#include <unistd.h>
#include <dirent.h>
#include <sys/stat.h>

char source_dir[1024];

// fungsi untuk baca 1.txt - 7.txt, cari line KOORD, gabungin fungsi generate_tujuan_content
    char fragment[1024] = "";
    char line[256];

    // baca 1.txt - 7.txt
    for (int i = 1; i <= 7; i++) {
        char filepath[4096];
        snprintf(filepath, sizeof(filepath), "%s/%d.txt", source_dir, i);

        FILE *f = fopen(filepath, "r");
        if (f) {
            while (fgets(line, sizeof(line), f)) {
                // cari line KOORD
                if (strncmp(line, "KOORD: ", 7) == 0) {
                    line[strcspn(line, "\r\n")] = 0;

                    strncat(fragment, line + 7, sizeof(fragment) - strlen(fragment) - 1);
                    break; // Lanjut ke file berikutnya
                }
            }
            fclose(f);
        }
    }
    snprintf(output_buffer, buf_size, "Tujuan Mas Amba: %s\n", fragment);
}


// FUSE

// Getattr
static int xmp_getattr(const char *path, struct stat *stbuf, struct fuse_file_info *fi) {
    (void) fi;
    int res = 0;
    memset(stbuf, 0, sizeof(struct stat));

    // kalo file virtual
    if (strcmp(path, "/tujuan.txt") == 0) {
        stbuf->st_mode = S_IFREG | 0444;
        stbuf->st_nlink = 1;

        char content[4096];
        generate_tujuan_content(content, sizeof(content));
        stbuf->st_size = strlen(content);

        stbuf->st_uid = getuid();
        stbuf->st_gid = getgid();
        return 0;
    }

    // kalo bukan file virtual -> passthrough
    char fpath[4096];
    snprintf(fpath, sizeof(fpath), "%s%s", source_dir, path);
    res = lstat(fpath, stbuf);
    if (res == -1) return -errno;

    return 0;
}

// Readdir
static int xmp_readdir(const char *path, void *buf, fuse_fill_dir_t filler,
                       off_t offset, struct fuse_file_info *fi,
                       enum fuse_readdir_flags flags) {
    (void) offset;
    (void) fi;
    (void) flags;

    char fpath[4096];
    if (strcmp(path, "/") == 0) {
        snprintf(fpath, sizeof(fpath), "%s", source_dir);
    } else {
        snprintf(fpath, sizeof(fpath), "%s%s", source_dir, path);
    }

    DIR *dp = opendir(fpath);
    if (dp == NULL) return -errno;

    struct dirent *de;
    while ((de = readdir(dp)) != NULL) {
        struct stat st;
        memset(&st, 0, sizeof(st));
        st.st_ino = de->d_ino;
        st.st_mode = de->d_type << 12;
        if (filler(buf, de->d_name, &st, 0, 0)) break;
    }
    closedir(dp);

    if (strcmp(path, "/") == 0) {
        filler(buf, "tujuan.txt", NULL, 0, 0);
    }

    return 0;
}

// Open file virtual
static int xmp_open(const char *path, struct fuse_file_info *fi) {
    if (strcmp(path, "/tujuan.txt") == 0) {
        return 0;
    }

    char fpath[4096];
    snprintf(fpath, sizeof(fpath), "%s%s", source_dir, path);
    int res = open(fpath, fi->flags);
    if (res == -1) return -errno;

    close(res);
    return 0;
}

// Read
static int xmp_read(const char *path, char *buf, size_t size, off_t offset,
                    struct fuse_file_info *fi) {
    (void) fi;

    if (strcmp(path, "/tujuan.txt") == 0) {
        char content[4096];
        generate_tujuan_content(content, sizeof(content));
        size_t len = strlen(content);

        if (offset < len) {
            if (offset + size > len) {
                size = len - offset;
            }
            memcpy(buf, content + offset, size);
        } else {
            size = 0;
        }
        return size;
    }

    char fpath[4096];
    snprintf(fpath, sizeof(fpath), "%s%s", source_dir, path);
    int fd = open(fpath, O_RDONLY);
    if (fd == -1) return -errno;

    int res = pread(fd, buf, size, offset);
    if (res == -1) res = -errno;

    close(fd);
    return res;
}

static struct fuse_operations xmp_oper = {
    .getattr = xmp_getattr,
    .readdir = xmp_readdir,
    .open    = xmp_open,
    .read    = xmp_read,
};

// Main Funct

int main(int argc, char *argv[]) {
    if (argc < 3) {
        fprintf(stderr, "Usage: %s <source_dir> <mount_dir>\n", argv[0]);
        return 1;
    }

    realpath(argv[1], source_dir);

    char *fuse_argv[2];
    fuse_argv[0] = argv[0];
    fuse_argv[1] = argv[2];

    return fuse_main(2, fuse_argv, &xmp_oper, NULL);
}
```


