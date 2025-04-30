## TASK 4

**Pipip's Load Balancer**
Task 4 : Pipip's Load Balancer meminta untuk membuat sistem distribusi pesan dengan sistem load balancing. Sistem dirancang untuk memungkinkan pesan dari client bisa disalurkan secara efisien ke beberapa worker. Dengan menggunakan komunikasi antar-proses (`IPC`)dan memastikan bahwa proses pengiriman pesan berjalan mulus dan terorganisir dengan baik, melalui sistem log yang tercatat dengan rapi.

### A : Client Mengirimkan Pesan ke Load Balancer

Membuat proses `client.c` agar dapat dapat mengirimkan pesan ke `loadbalancer.`c menggunakan IPC dengan metode shared memory. Selain itu, setiap kali pesan dikirim, proses client.c harus menuliskan aktivitasnya ke dalam `sistem.log`

**Code**
client.c 
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <unistd.h>

int main(){
    char input[512], message[512];
    int count;

    printf("input [ message;sum ] : ");
    fgets(input, sizeof(input), stdin);
    sscanf(input, "%[^;];%d", message, &count);

    char full[1024];
    snprintf(full, sizeof(full), "%s\n%d\n", message, count);

    int shid = shmget(1234, 1024, IPC_CREAT | 0666);
    char* shP = (char*) shmat(shid, NULL, 0);

    while (shP[0] != '\0') sleep(1);

    strcpy(shP, full);

    FILE* log = fopen("sistem.log", "a");
    fprintf(log, "Message from client: %s\n", message);
    fprintf(log, "Message count: %d\n", count);
    fclose(log);

    shmdt(shP);
    return 0;
}
```

**Penjelasan :**
- `stdio.h`, `stdlib.h`, `string.h` : Input/output standar, alokasi memori, dan manipulasi string.
- `sys/ipc.h`, `sys/shm.h` : Untuk menggunakan fungsi-fungsi IPC dengan shared memory.
- `unistd.h` : Digunakan untuk fungsi `sleep()`.

```
char input[512], message[512];
int count;

printf("input [ message;sum ] : ");
fgets(input, sizeof(input), stdin);
sscanf(input, "%[^;];%d", message, &count);
```
- `fgets` membaca input dengan syarat : message;sum
- `sscanf` memecah input menjadi dua bagian : `message` (sebelum titik koma) dan `count` (setelah titik koma)

```
char full[1024];
snprintf(full, sizeof(full), "%s\n%d\n", message, count);
```
- Menggabungkan message dan count menjadi satu string dan dipisahkan oleh newline (`\n`), untuk dikirim ke shared memory.

```
int shid = shmget(1234, 1024, IPC_CREAT | 0666);
char* shP = (char*) shmat(shid, NULL, 0);
```
- `shmget` membuat atau mendapatkan shared memory segment dengan **key = 1234** dan **ukuran = 1024 byte**.
- `shmat` meng-attach segment tersebut ke alamat memori proses dan mengembalikan pointer-nya (`shP`).

```
while (shP[0] != '\0') sleep(1);
strcpy(shP, full);
```
- Menunggu sampai karakter pertama di shared memory adalah null (`'\0'`).
- Menyalin isi `full` ke shared memory sehingga bisa dibaca oleh proses `loadbalancer.c`.

```
FILE* log = fopen("sistem.log", "a");
fprintf(log, "Message from client: %s\n", message);
fprintf(log, "Message count: %d\n", count);
fclose(log);
```
- Mencatat isi pesan dan jumlah pesan ke file `sistem.log`.
- File dibuka dengan mode append ("a"), jadi data baru ditambahkan di akhir file.

```
shmdt(shP);
```
- Memutus koneksi (detach) pointer `shP` dari shared memory.

**Ouput/input :**

client.c*
```
input [ message;sum ] : halo;5
```

sistem.log
```
Message from client: halo
Message count: 5
```

**Sample Output :** <br>
![Screenshot 2025-04-30 172005](https://github.com/user-attachments/assets/511c5f2a-0bfc-4d7f-bb0d-c1ce2cf3af68) <br>
![image](https://github.com/user-attachments/assets/f9e210b2-8e6d-41c5-93f6-10ac075069ea)
---


### B : Load Balancer Mendistribusikan Pesan ke Worker Secara Round-Robin

Setelah menerima pesan dari `client.c`, `loadbalancer.c` akan mendistribusikan pesan-pesan tersebut ke beberapa worker menggunakan metode round-robin. Sebelum mendistribusikan pesan, `loadbalancer.c` terlebih dahulu mencatat informasi ke dalam `sistem.log`.

**Code :**
loadbalancer.c
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <sys/msg.h>
#include <unistd.h>

struct msgbuf{
    long type;
    char text[1024];
};

int main(){
    int srid = shmget(1234, 1024, 0666);
    if (srid < 0){
        perror("shmget failed");
        exit(1);
    }

    char* srp = (char*) shmat(srid, NULL, 0);
    if (srp == (char *) -1){
        perror("shmat failed");
        exit(1);
    }

    int msid = msgget(5678, IPC_CREAT | 0666);
    if (msid < 0){
        perror("msgget failed");
        exit(1);
    }

    while (1){
        if (srp[0] != '\0'){
            char text[512];
            int count;

            if (sscanf(srp, "%[^\n]\n%d", text, &count) != 2){
                fprintf(stderr, "Format input tidak valid. Gunakan format: message;jumlah\n");
                srp[0] = '\0';
                continue;
            }

            FILE* log = fopen("sistem.log", "a");
            for (int i = 0; i < count; i++){
                fprintf(log, "Received at lb: %s (#message %d)\n", text, i + 1);
\
                struct msgbuf msg;
                msg.type = (i % 3) + 1;
                strcpy(msg.text, text);
                if (msgsnd(msid, &msg, strlen(msg.text) + 1, 0) == -1){
                    perror("failed");
                    exit(1);
                }
            }
            fclose(log);
            
            srp[0] = '\0';
        }

        sleep(1); 
    }

    shmdt(srp);
    return 0;
}
```

**Penjelasan :**
- `stdio.h`, `stdlib.h`, `string.h` : Input/output standar, alokasi memori, dan manipulasi string.
- `sys/ipc.h`, `sys/shm.h` : Untuk menggunakan fungsi-fungsi IPC dengan shared memory.
- `unistd.h` : Digunakan untuk fungsi `sleep()`.

```
struct msgbuf{
    long type;
    char text[1024];
};
```
- `typer` : Menentukan worker yang akan menerima pesan
- `text` : Isi pesan

```
int srid = shmget(1234, 1024, 0666);
```
- Mendapatkan ID shared memory dengan key **1234** dan ukuran **1024** byte.
- Mode `0666` : Read/write untuk user, group, others.

```
int msid = msgget(5678, IPC_CREAT | 0666);
```
- Membuat atau mengambil message queue dengan key `5678`.
- Mode akses read/write.

```
while (1) {
    if (srp[0] != '\0') {
        // the rest
    }
    sleep(1);
}
```
- Loop terus berjalan.
- Mengecek apakah ada pesan baru di shared memory `(srp[0] != '\0')`.
- Setelah selesai diproses, `srp[0] = '\0'` agar pesan tidak dikirim dua kali.
- `sleep` 1 detik agar tidak membebani CPU.

```
if (sscanf(srp, "%[^\n]\n%d", text, &count) != 2)
```
- Mengekstrak teks dan jumlah pengiriman dari shared memory.
- Format input harus seperti: `pesan\njumlah`
- Jika format salah, tampilkan error dan kosongkan shared memory.

```
for (int i = 0; i < count; i++) {
    fprintf(log, "Received at lb: %s (#message %d)\n", text, i + 1);

    struct msgbuf msg;
    msg.type = (i % 3) + 1;  // Untuk mengirim ke worker 1, 2, atau 3
    strcpy(msg.text, text);

    msgsnd(msid, &msg, strlen(msg.text) + 1, 0);
}
```
- Menulis log untuk setiap pesan yang diterima.
- `msg.type = (i % 3) + 1` : membagi pesan ke 3 worker secara merata.
- `msgsnd` : mengirim pesan ke message queue.

```
shmdt(srp);
```
- Detach pointer dari shared memory (meskipun tidak pernah dieksekusi karena `while(1)`).

**Ouput/input :**

sistem.log
```
Received at lb: halo (#message 1)
Received at lb: halo (#message 2)
Received at lb: halo (#message 3)
Received at lb: halo (#message 4)
Received at lb: halo (#message 5)
```
tidak ada output yang ditampilkan di terminal pada `loadbalancer.c`

**Sample Output :** <br>
![Screenshot 2025-04-30 172122](https://github.com/user-attachments/assets/de2b1c76-5cf8-4ec7-869b-2a46066b78e1) <br>
![image](https://github.com/user-attachments/assets/b09dcd47-0743-4933-a609-eb0f0993bdc4)
---


### C : Worker Mencatat Pesan yang Diterima

Setiap worker yang menerima pesan dari loadbalancer.c harus mencatat pesan yang diterima ke dalam `sistem.log`

**Code :**
worker.c
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <unistd.h>
#include <signal.h>

struct buf {
    long type;
    char text[1024];
};

int id;
int total = 0;

void handle(int sig){
    FILE *log = fopen("sistem.log", "a");
    if (log != NULL) {
        fprintf(log, "Worker %d: %d messages\n", id, total);
        fclose(log);
    }
    printf("Total messages: %d\n",  total);
    exit(0);
}

int main(int argc, char *argv[]){
    if (argc != 2) {
        fprintf(stderr, "Cara pakai: %s <nomor_worker>\n", argv[0]);
        exit(1);
    }

    id = atoi(argv[1]);

    signal(SIGINT, handle);

    int IDmsg = msgget(5678, 0666);
    if (IDmsg < 0) {
        perror("failed");
        exit(1);
    }

    struct buf buff;
    printf("Worker %d ready.\n", id);

    while (1) {
        if (msgrcv(IDmsg, &buff, sizeof(buff.text), id, 0) != -1) {
            printf("Worker %d: %s\n", id, buff.text);

            FILE *log = fopen("sistem.log", "a");
            if (log != NULL) {
                fprintf(log, "Worker%d: message received\n", id);
                fclose(log);
            }

            total++;
        }
    }

    return 0;
}
```

**Penjelasan :**
- Standar input/output dan memori (stdio, stdlib).
- String handling.
- Fungsi IPC untuk message queue.
- Signal handling (SIGINT → fungsi handle).

```
struct buf {
    long type;
    char text[1024];
};
```
- `typer` : Menentukan worker yang akan menerima pesan
- `text` : Isi pesan

```
int id;
int total = 0;
```
- `id` : ID/nama worker.
- `total` : jumlah pesan yang telah diterima oleh worker.

```
void handle(int sig){
    FILE *log = fopen("sistem.log", "a");
    if (log != NULL) {
        fprintf(log, "Worker %d: %d messages\n", id, total);
        fclose(log);
    }
    printf("Total messages: %d\n",  total);
    exit(0);
}
```
- Ketika user menekan `Ctrl+C`, fungsi ini:
    - Menuliskan jumlah pesan ke `sistem.log`.
    - Menampilkan jumlah pesan ke terminal.
    - Keluar dari program.

```
if (argc != 2) {
    fprintf(stderr, "Cara pakai: %s <nomor_worker>\n", argv[0]);
    exit(1);
}
id = atoi(argv[1]);
```
- Mengecek argumen saat menjalankan program.
- ID worker harus diberikan, misal: `./worker 1`

```
signal(SIGINT, handle);
```
- Menghubungkan `SIGINT` (dari `Ctrl+C`) ke fungsi handle.

```
int IDmsg = msgget(5678, 0666);
```
- Terhubung ke message queue yang sudah dibuat oleh `Load Balancer`

```
while (1) {
    if (msgrcv(IDmsg, &buff, sizeof(buff.text), id, 0) != -1) {
        // the rest
    }
}
```
- Worker akan terus menunggu pesan dengan type == id.
- Ketika pesan diterima maka akan ditampilkan ke terminal, Dicatat di `sistem.log` dan Variabel total ditambah.

**Ouput/Input :**

worker.c : 
./worker (n) -> example
```
./worker n
Worker n ready.
Worker n: [message]
.......
^CWorker 1 exiting. Total messages: m
```

**Sample Output** <br>
![Screenshot 2025-04-30 172107](https://github.com/user-attachments/assets/8664e0d6-c3ca-4128-85b0-3ef1d5e7b7f7) <br>
![Screenshot 2025-04-30 172148](https://github.com/user-attachments/assets/9a6ab33a-db75-451f-9e48-6ce1ad8364e0) <br>
![Screenshot 2025-04-30 172214](https://github.com/user-attachments/assets/df6d80ed-3815-45cc-bc63-e4d714067bd1)
---


### D : Catat Total Pesan yang Diterima Setiap Worker di Akhir Eksekusi

Setelah proses selesai (semua pesan sudah diproses), setiap worker akan mencatat jumlah total pesan yang mereka terima ke bagian akhir file `sistem.log`.

**sistem.log Ouput :**
```
Message from client: halo
Message count: 5
Received at lb: halo (#message 1)
Received at lb: halo (#message 2)
Received at lb: halo (#message 3)
Received at lb: halo (#message 4)
Received at lb: halo (#message 5)
Worker3: message received
Worker1: message received
Worker1: message received
Worker2: message received
Worker2: message received
Worker 1: 2 messages
Worker 2: 2 messages
Worker 3: 1 messages
```

**Ouput Sample :** <br>
![image](https://github.com/user-attachments/assets/ef7557b7-d0c5-4503-ba01-ac6cc02be2ed)
---


### HOW TO RUN THE CODE
- Jalankan semua proses di terminal berbeda 
- Jalankan `loadbalancer.c` terlebih dahulu
- Setelahnya jalankan `worker.c` sesuai dengan jumlah worker yang ada dan setiap worker di run pada terminal yang berbeda
- Terakhir baru jalankan `client.c` dan input pesan dan jumlah pesan yang diinginkan
- Jika pesan sudah input maka input `Ctrl+C` kepada semua proses yang berjalan untuk menghentikan program dan hasil dicatat pada `sistem.log`
