### Soal 4

> Dosen meminta Budiman membuat sistem operasi ini memilki **superuser** layaknya sistem operasi pada umumnya. User root yang sudah kamu buat sebelumnya akan digunakan sebagai superuser dalam sistem operasi milik Budiman. Superuser yang dimaksud adalah user dengan otoritas penuh yang dapat mengakses seluruhnya. Akan tetapi user lain tidak boleh memiliki otoritas yang sama. Dengan begitu user-user selain root tidak boleh mengakses `./root`. Buatlah sehingga tiap user selain superuser tidak dapat mengakses `./root`!

> _The lecturer requests that the OS must have a **superuser** just like other operating systems. The root user created earlier will serve as the superuser in Budiman's OS. The superuser should have full authority to access everything. However, other users should not have the same authority. Therefore, users other than root should not be able to access `./root`. Implement this so that non-superuser accounts cannot access `./root`!_

**Answer:**

- **Code:**

  chmod 700 root

- **Explanation:**

  Melakukan perintah `chmod 700 root` agar hanya user **root** (superuser) yang dapat membuka, membaca, atau menulis ke direktori `./root` sehingga user lain, meskipun berada dalam sistem, akan mendapatkan pesan ‘`Permission denied`’ jika mencoba mengakses direktori tersebut.

- **Screenshot:**

  ![Screenshot 2025-05-28 185414](https://github.com/user-attachments/assets/a5396898-5c8f-4d69-ab70-0face39fd9ec)
  ![Screenshot 2025-05-23 205427](https://github.com/user-attachments/assets/5e37e156-917c-4382-a16f-90e925cb4aba)

### Soal 6

> Dosen Budiman menginginkan sistem operasi yang **stylish**. Budiman memiliki ide untuk membuat sistem operasinya menjadi stylish. Ia meminta kamu untuk menambahkan tampilan sebuah banner yang ditampilkan setelah suatu user login ke dalam sistem operasi Budiman. Banner yang diinginkan Budiman adalah tulisan `"Welcome to OS'25"` dalam bentuk **ASCII Art**. Buatkanlah banner tersebut supaya Budiman senang! (Hint: gunakan text to ASCII Art Generator)

> _Budiman wants a **stylish** operating system. Budiman has an idea to make his OS stylish. He asks you to add a banner that appears after a user logs in. The banner should say `"Welcome to OS'25"` in **ASCII Art**. Use a text to ASCII Art generator to make Budiman happy!_ (Hint: use a text to ASCII Art generator)

**Answer:**

- **Code:**

  etc/motd : Input hasil generate ASCII art generator ke dalam `etc/motd`

- **Explanation:**

  - Membuat sebuah file bernama `motd` di dalam direktori `etc/`
  - Input hasil generate ASCII art generator ke dalam `etc/motd`

- **Screenshot:**

  ![Screenshot 2025-05-28 211120](https://github.com/user-attachments/assets/63ed84dc-1b76-4d79-a309-431f94b0e537)


### Soal 7

> Melihat perkembangan sistem operasi milik Budiman, Dosen kagum dengan adanya banner yang telah kamu buat sebelumnya. Kemudian Dosen juga menginginkan sistem operasi Budiman untuk dapat menampilkan **kata sambutan** dengan menyebut nama user yang login. Sambutan yang dimaksud berupa kalimat `"Helloo %USER"` dengan `%USER` merupakan nama user yang sedang menggunakan sistem operasi. Kalimat sambutan ini ditampilkan setelah user login dan setelah banner. Budiman kembali lagi meminta bantuanmu dalam menambahkan fitur ini.

> _Seeing the progress of Budiman's OS, the lecturer is impressed with the banner you created. The lecturer also wants the OS to display a **greeting message** that includes the name of the user who logs in. The greeting should say `"Helloo %USER"` where `%USER` is the name of the user currently using the OS. This greeting should be displayed after user login and after the banner. Budiman asks for your help again to add this feature._

**Answer:**

- **Code:**

  etc/profile
  ```
  #!/bin/sh
  echo "Helloo $USER"
  ```

- **Explanation:**

  - Membuat file `profile` di dalam path `etc/` dari lokasi path `myramdisk`
  - Lalu mengisi file tersebut denga kode diatas.
  - Variabel `USER` pada kode tersebut akan mengembalikan nama pengguna yang berhasil melakukan proses login.


- **Screenshot:**

  ![Screenshot 2025-05-23 210036](https://github.com/user-attachments/assets/284c0100-d174-40ab-a093-6bed07144d16)


### Soal 8

> Dosen Budiman sudah tua sekali, sehingga beliau memiliki kesulitan untuk melihat tampilan terminal default. Budiman menginisiatif untuk membuat tampilan sistem operasi menjadi seperti terminal milikmu. Modifikasilah sistem operasi Budiman menjadi menggunakan tampilan terminal kalian.

> _Budiman's lecturer is quite old and has difficulty seeing the default terminal display. Budiman takes the initiative to make the OS look like your terminal. Modify Budiman's OS to use your terminal appearance!_

**Answer:**

- **Code:**

  myramdisk/init
    ```
    #!/bin/sh
    /bin/mount -t proc none /proc
    /bin/mount -t sysfs none /sys

    while true
    do
        /bin/getty -L ttyS0 115200 vt100
        sleep 1
    done
    ```
  linux-6.1.1/.config
  ```
  CONFIG_SERIAL_8250=y
  ```
  qemu
  ```
  qemu-system-x86_64 \
  -smp 2 \
  -m 256 \
  -nographic \
  -kernel bzImage \
  -initrd myramdisk.gz \
  -append "console=ttyS0"
  ```

- **Explanation:**

  - Buka `myramdisk/init` di mousepad lalu jalankan perintah diatas.
  - Buka `linux-6.1.1/.config` lalu ubah `CONFIG_SERIAL_8250` yang belum di set menjadi **`CONFIG_SERIAL_8250=y`**.
  - Terkahir program bisa di jalankan dengan perintah `qemu` diatas.

- **Screenshot:**

  ![Screenshot 2025-05-23 211009](https://github.com/user-attachments/assets/d300b084-6052-4b0e-81c2-ce0cd15d0d6b)
