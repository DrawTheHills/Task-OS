# LATIHAN MODUL 2

## IDENTITAS 
- NRP      : `5025241246`
- Nama     : `Naufal Fadhlil Wafi`
- Kelas    : `Sistem Operasi A`
- Kelompok : `A11`

### TASK 1
program.c
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <string.h>

int main() {
    const char *folderName = "halo";
    const char *fileInside = "halo/hai.txt";
    const char *fileOutside = "hai.txt";

    // Halo
    if (mkdir(folderName, 0755) == -1) {
        perror("Gagal membuat folder");
    } else {
        printf("Folder '%s' berhasil dibuat.\n", folderName);
    }

    // "halo/hai.txt"
    FILE *file = fopen(fileInside, "w");
    if (file == NULL) {
        perror("Gagal membuat file");
        return 1;
    }
    fclose(file);
    printf("File '%s' berhasil dibuat.\n", fileInside);

    FILE *src = fopen(fileInside, "r");
    FILE *dst = fopen(fileOutside, "w");

    if (src == NULL || dst == NULL) {
        perror("Gagal membuka file");
        return 1;
    }

    char ch;
    while ((ch = fgetc(src)) != EOF) {
        fputc(ch, dst);
    }

    fclose(src);
    fclose(dst);

    printf("File berhasil disalin ke '%s'.\n", fileOutside);

    return 0;
}
```
**Penjelasan** 
Program.c melakukan 3 hal secara bersamaan :
- Membuat folder halo
- Membuat file kosong halo/hai.txt
- Menyalin halo/hai.txt ke ./hai.txt

**Ouput**
Hasil Ouput :
```
Folder 'halo' berhasil dibuat.
File 'halo/hai.txt' berhasil dibuat.
File berhasil disalin ke 'hai.txt'.
```

**Screenshot**<br>
![Screenshot 2025-04-20 104717](https://github.com/user-attachments/assets/ebaf1d98-8b33-424e-9269-08a43200e740)


### TASK 2
progarm2.c
```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>

pthread_mutex_t lock;
int finish_order[3];
int order_index = 0;

void write_done(int thread_num) {
    pthread_mutex_lock(&lock);
    if (order_index < 3) {
        finish_order[order_index++] = thread_num;
    }
    pthread_mutex_unlock(&lock);
}

void* thread1_func(void* arg) {
    FILE* f = fopen("count.txt", "w");
    for (int i = 1; i <= 100; ++i) {
        fprintf(f, "%d\n", i);
    }
    fclose(f);
    write_done(1);
    pthread_exit(NULL);
}

void* thread2_func(void* arg) {
    FILE* f = fopen("print.txt", "w");
    fprintf(f, "Saya pintar mengerjakan thread\n");
    fclose(f);
    write_done(2);
    pthread_exit(NULL);
}

void* thread3_func(void* arg) {
    FILE* f = fopen("count_2.txt", "w");
    for (int i = 2; i <= 100; i += 2) {
        fprintf(f, "%d\n", i);
    }
    fclose(f);
    write_done(3);
    pthread_exit(NULL);
}

int main() {
    pthread_t t1, t2, t3;
    pthread_mutex_init(&lock, NULL);

    pthread_create(&t1, NULL, thread1_func, NULL);
    pthread_create(&t2, NULL, thread2_func, NULL);
    pthread_create(&t3, NULL, thread3_func, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);
    pthread_join(t3, NULL);

    FILE* log = fopen("log.txt", "a");
    for (int i = 0; i < 3; ++i) {
        fprintf(log, "%d. Thread %d\n", i + 1, finish_order[i]);
    }
    fclose(log);

    pthread_mutex_destroy(&lock);
    return 0;
}
```
**Penjelasan**
Membuat sebuah program C yang menggunakan tiga buah thread untuk menjalankan tiga proses berbeda secara bersamaa : 
- *Thread 1* bertugas menuliskan angka dari 1 hingga 100 secara berurutan ke dalam sebuah file bernama `count.txt`
- *Thread 2* bertugas menuliskan sebuah kalimat statis, yaitu "Saya pintar mengerjakan thread" ke dalam file bernama `print.txt`
- *Thread 3* bertugas menuliskan semua angka genap dari 1 hingga 100 ke dalam file bernama `count_2.txt`

**Output**
```
1. Thread 2
2. Thread 1
3. Thread 3
1. Thread 2
2. Thread 1
3. Thread 3
1. Thread 1
2. Thread 2
3. Thread 3
```
**Screenshot**<br>
![Screenshot 2025-04-20 115409](https://github.com/user-attachments/assets/d5af7ee6-1519-4088-8931-f9356986636a)


### TASK 3
program3.c
```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>

#define THREAD_COUNT 5

pthread_mutex_t lock;

void* count_function(void* arg) {
    int id = *((int*) arg);

    pthread_mutex_lock(&lock);

    FILE* file = fopen("log.txt", "a");
    if (file == NULL) {
        perror("Gagal membuka file");
        pthread_mutex_unlock(&lock);
        pthread_exit(NULL);
    }

    for (int i = 1; i <= 3; i++) {
        fprintf(file, "thread %d count %d\n", id, i);
        fflush(file);
        sleep(1);    
    }

    fclose(file);
    pthread_mutex_unlock(&lock);
    pthread_exit(NULL);
}

int main() {
    pthread_t threads[THREAD_COUNT];
    int ids[THREAD_COUNT];

    pthread_mutex_init(&lock, NULL);

    for (int i = 0; i < THREAD_COUNT; i++) {
        ids[i] = i + 1;
        pthread_create(&threads[i], NULL, count_function, &ids[i]);
    }

    for (int i = 0; i < THREAD_COUNT; i++) {
        pthread_join(threads[i], NULL);
    }

    pthread_mutex_destroy(&lock);
    return 0;
}
```

**Penjelasan**
Membuat sebuah program dalam bahasa C yang menggunakan konsep multithreading dengan bantuan pustaka `pthread`. Program ini harus membuat lima buah thread, di mana setiap thread akan menjalankan tugas yang sama, yaitu :
- Melakukan perhitungan dari angka 1 sampai 3
- Menuliskan hasil perhitungan tersebut ke dalam sebuah file bernama `log.txt`

**Output**
```
thread 2 count 1
thread 2 count 2
thread 2 count 3
thread 1 count 1
thread 1 count 2
thread 1 count 3
thread 3 count 1
thread 3 count 2
thread 3 count 3
thread 4 count 1
thread 4 count 2
thread 4 count 3
thread 5 count 1
thread 5 count 2
```

**Screenshot**<br>
![Screenshot 2025-04-20 111253](https://github.com/user-attachments/assets/41dd465b-2056-4540-9b59-7d4aa2bf54d5)


### TASK 4

#### TASK 4.A
**sender.c**
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <string.h>

#define SHM_SIZE 1024 

int main() {
    key_t key = 1234;  
    int shm_id;
    char *shm_ptr;

    shm_id = shmget(key, SHM_SIZE, 0666 | IPC_CREAT);
    if (shm_id == -1) {
        perror("shmget failed");
        exit(1);
    }

    shm_ptr = (char *)shmat(shm_id, NULL, 0);
    if (shm_ptr == (char *)-1) {
        perror("shmat failed");
        exit(1);
    }

    strcpy(shm_ptr, "aku lagi belajar ipc");

    printf("Pesan dikirim ke shared memory: %s\n", shm_ptr);

    printf("Menunggu receiver membaca pesan...\n");

    shmdt(shm_ptr);

    return 0;
}
```

**receiver.c**
```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/ipc.h>
#include <sys/shm.h>

#define SHM_SIZE 1024  

int main() {
    key_t key = 1234; 
    int shm_id;
    char *shm_ptr;

    shm_id = shmget(key, SHM_SIZE, 0666);
    if (shm_id == -1) {
        perror("shmget failed");
        exit(1);
    }

    shm_ptr = (char *)shmat(shm_id, NULL, 0);
    if (shm_ptr == (char *)-1) {
        perror("shmat failed");
        exit(1);
    }

    printf("Pesan yang diterima dari shared memory: %s\n", shm_ptr);

    printf("Receiver selesai membaca pesan.\n");

    shmdt(shm_ptr);

    return 0;
}
```
**Penjelasan**
Membuat dua buah program dalam bahasa C yang menggunakan metode IPC berbasis Shared Memory
- `shared,c`   :  Membuat shared memory dan mengirim pesan `aku lagi belajar ipc`
- `receiver.c` :  Membaca pesan dari shared memory dan menampilkannya ke layar

**Ouput**

./sender :
```
Pesan dikirim ke shared memory: aku lagi belajar ipc
Menunggu receiver membaca pesan...
```
./receiver :
```
Pesan yang diterima dari shared memory: aku lagi belajar ipc
Receiver selesai membaca pesan.
```

**Screenshot**<br>
![Screenshot 2025-04-20 113249](https://github.com/user-attachments/assets/8ea2bd54-7e6a-4095-9e52-59e04aca2c71)


### TASK 4.B
mq_sender.c
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ipc.h>
#include <sys/msg.h>

struct msg_buffer {
    long msg_type;
    char msg_text[100];
};

int main() {
    key_t key = 1234;
    int msgid;

    msgid = msgget(key, 0666 | IPC_CREAT);
    if (msgid == -1) {
        perror("msgget failed");
        exit(1);
    }

    struct msg_buffer message;
    message.msg_type = 1;
    strcpy(message.msg_text, "yah belajar ipc mulu");

    if (msgsnd(msgid, &message, sizeof(message.msg_text), 0) == -1) {
        perror("msgsnd failed");
        exit(1);
    }

    printf("Pesan dikirim: %s\n", message.msg_text);
    return 0;
}
```

mq_receiver.c
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ipc.h>
#include <sys/msg.h>

struct msg_buffer {
    long msg_type;
    char msg_text[100];
};

int main() {
    key_t key = 1234;
    int msgid;

    msgid = msgget(key, 0666);
    if (msgid == -1) {
        perror("msgget failed");
        exit(1);
    }

    struct msg_buffer message;

    if (msgrcv(msgid, &message, sizeof(message.msg_text), 1, 0) == -1) {
        perror("msgrcv failed");
        exit(1);
    }

    printf("Pesan diterima: %s\n", message.msg_text);

    msgctl(msgid, IPC_RMID, NULL);

    return 0;
}
```

**Penjelasan**
Membuat program C untuk komunikasi antar proses menggunakan Message Queue :
- `mq_sender.c`   : Mengirim pesan ke message queue
- `mq_receiver.c` : Menerima pesan dari message queue

**Ouput**
./mqsender :
```
Pesan dikirim: yah belajar ipc mulu
```
./mqreceiver :
```
Pesan diterima: yah belajar ipc mulu
```

**Screenshot**<br>
![Screenshot 2025-04-20 114128](https://github.com/user-attachments/assets/82312d5a-d8a4-4753-9355-f64d6731a8e2)


### TASK 4.C
program4.c
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

int main() {
    int fd[2]; 
    pid_t pid;
    char message[] = "hai, anak sisop 24";
    char buffer[100];

    if (pipe(fd) == -1) {
        perror("Pipe gagal dibuat");
        exit(EXIT_FAILURE);
    }

    pid = fork();

    if (pid < 0) {
        perror("Fork gagal");
        exit(EXIT_FAILURE);
    }

    if (pid > 0) {
        close(fd[0]); 
        write(fd[1], message, strlen(message) + 1); 
        close(fd[1]); 
    } else {
        close(fd[1]);
        read(fd[0], buffer, sizeof(buffer));
        printf("Child menerima pesan: %s\n", buffer);
        close(fd[0]); /
    }

    return 0;
}
```

**Penjelasan**
Membuat sebuah program C yang menggunakan mekanisme IPC melalui pipe dan fungsi `fork()` :
- Menggunakan pipe dan `fork()` untuk membuat `child process`.
- `Parent process` mengirim string: hai, anak sisop 24

**Ouput**
```
Child menerima pesan: hai, anak sisop 24
```

**Screenshot**<br>
![Screenshot 2025-04-20 114640](https://github.com/user-attachments/assets/44d69564-3764-4e15-a87e-eb26e92badb0)
