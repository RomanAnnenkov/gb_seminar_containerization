# Контейнеризация (семинары)

## Урок 1. Механизмы пространства имен

### Изоляция на уровне файловой системы - chroot
Утилита **_chroot_** запускает команду или интерактивный терминал в указанной корневой директории.

- создадим директорию _testfolder_, которую будем передавать chroot в качестве параметра

    `mkdir ~/testfolder`

- создадим директорию _bin_, для исполняемых файлов

    `mkdir ~/testfolder/bin`

- скопируем исполняемый файл командной оболочки в директорию _bin_

    `cp /bin/bash ~/testfolder/bin/`

    Сам по себе исполняемый файл работать не сможет, поэтому помимо файла нужно еще скопировать необходимые ему библиотеки.

- для просмотра зависимостей воспользуемся утилитой **ldd**

    `ldd /bin/bash` 

    ![ldd output bash](source/ldd_output_bash.png)

- создадим директории и скопируем зависимости

    ```
    mkdir ~/testfolder/lib ~/testfolder/lib64
    cp /lib/x86_64-linux-gnu/libtinfo.so.6 ~/testfolder/lib/
    cp /lib/x86_64-linux-gnu/libc.so.6 ~/testfolder/lib/
    cp /lib64/ld-linux-x86-64.so.2 ~/testfolder/lib64/
    ```

- после копирования необходимых файлов запустим **chroot**

    `sudo chroot ~/testfolder`

    ![chroot only bash](source/chroot_only_bash.png)

    Т.к. мы скопировали только **bash** вместе с зависимостями, мы не можем использовать другие утилиты.

- для добавления утилиты **ls**, посмотрим какие у неё зависимости с помощью **ldd**

    `ldd /bin/ls`

    ![ldd output ls](source/ldd_output_ls.png)

- скопируем файлы

    ```
    cp /bin/ls ~/testfolder/bin/
    cp /lib/x86_64-linux-gnu/libselinux.so.1 ~/testfolder/lib/
    cp /lib/x86_64-linux-gnu/libc.so.6 ~/testfolder/lib/
    cp /lib/x86_64-linux-gnu/libpcre2-8.so.0 ~/testfolder/lib/
    cp /lib64/ld-linux-x86-64.so.2 ~/testfolder/lib64/
    ```

- после того как необходимые файлы скопированы, запустим **chroot** и проверим **ls**

    `sudo chroot ~/testfolder`

    ![chroot bash ls](source/chroot_bash_ls.png)

    Теперь мы можем использовать **ls** внутри **chroot**, по такому же принципу, можно добавить все необходимые утилиты.

### Изоляция на сетевом уровне - ip netns

### Изоляция на уровне процессов и сетевом уровне - unshare
