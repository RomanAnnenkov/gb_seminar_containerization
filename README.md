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

Для управления сетевым пространством имён воспользуемся утилитой ip c параметром netns.

- для создания воспользуемся командой **ip netns add** и укажем имя

    `sudo ip netns add testns`

- после создания мы можем обращатся по имени к созданному пространству и передавать команды для исполнения

    `sudo ip netns exec testns bash`

    Эта команда запустит оболочку bash в указанном пространстве имён.

    ![netns bash](source/netns_bash.png)

    После выполения команды **ip a** мы видим только один сетевой интерфейс loopback, что говорит нам о успешной сетевой изоляции.


### Изоляция на уровне процессов и сетевом уровне - unshare

Утилита **unshare** позволяет нам с помощью параметров создавать разные уровни пространств имён и запускать в этом окружении переданную в качестве аргумента программу, если её не передать запустится **/bin/sh**

- в качестве примера используем изоляцию на уровне сети и процессов

    `sudo unshare --net --pid --fork --mount-proc /bin/bash`

    После запуска мы можем проверить изоляцию сети через команду **ip a**, а изоляцию процессов через **ps aux**.

    ![unshare net pid](source/unshare_net_pid.png)

    Мы видим только один сетевой интерфейс, что показывает изоляцию на уровне сети, и только процессы запущенные внутри пространства имён, что показывает изоляцию на уровне процессов.

## Урок 2. Механизмы контрольных групп

### Подготовка системы для работы с lxc

- Установим необходимые утилиты для работы с lxc

    `sudo apt install lxc lxc-utils`

### Создание конейнера
- Для создания контейнера воспользуемся утилитой **lxc-create**  с параметрами

    `sudo lxc-create -t download -n test_container -- -d ubuntu -r bionic -a amd64`

- Проверим что контейнер создан утилитой **lxc-ls**

    `sudo lxc-ls -f`

    ![lxc ls](source/lxc_ls_create_container.png)

- Запустим его с помощью утилиты **lxc-start**

    `sudo lxc-start -n test_container`

    ![lxc ls](source/lxc_ls_running_container.png)

- После того как контейнер запущен, можно подключиться к оболочке утилитой **lxc-attach**

    `sudo lxc-attach -n test_container`

    ![lxc attach](source/lxc_ls_attach_container.png)


### Ограничение ресурсов контейнера на примере оперативной памяти

- Для ограничения ресурсов контейнера, можно воспользоваться конфигурационным файлом.

    `sudo vim /var/lib/lxc/test_container/config`

- Добавив параметр отвечающий за ограничение оперативной памяти.

    ![lxc config](source/lxc_config_memory_limit.png)

- Для применения настроек из конфигурационного файла, нужно перезапустить контейнер.

    ```
    sudo lxc-stop -n test_container
    sudo lxc-start -n test_container
    ```

- Теперь проверим оперативную память доступную в контейнере.

    ![lxc check memory](source/lxc_check_memory_limit.png)

    Настройки из конфигурационного файла успешно применились.

### Настройка автозапуска контейнера

- Для включения автозапуска контейнера, нужно добавить соответствующий параметр в конфигурационный файл.

    ![lxc config](source/lxc_config_autostart.png)

- После перезапуска можно увидеть изменённый флаг в статусе контейнера.

    ![lxc autostart](source/lxc_autostart.png)

### Настройка логирования

- Для включения логирования нужно задать несколько параметров в конфигурационном файле.

    ![lxc log](source/lxc_config_logs.png)

- Параметром **_lxc.log.level_** определяется уровень подробности логирования. Варианты:
    - 0 = trace
    - 1 = debug 
    - 2 = info
    - 3 = notice
    - 4 = warn
    - 5 = error
    - 6 = critical
    - 7 = alert
    - 8 = fatal

    Значение по умолчанию для этого парметра  равно 5.

- Параметром **_lxc.log.file_** задаётся файл, в который будет происходить запись.

    Для применения настроек, контейнер нужно перезапустить, и можно следить за изменениями в лог файле.

    `sudo tail -f /var/log/lxc/test_container.log`

## Урок 3. Введение в Docker

### Подготовка системы для работы с docker

- Проверим наличие утилиты **_docker_** в системе.

    `docker --version`

    ![docker version](source/docker_version.png)

- Проверим что служба **_dockerd_** запущена и отвечает на запросы.

    `docker ps`

    ![docker ps](source/docker_ps.png)

- в случае если после выполнения предыдущих команд вы видите ошибку, то можно воспользоваться официальной документацией по установке docker для вашего дистрибутива (https://docs.docker.com/engine/install/). 

### Управление контейнерами и образами с помощью утилиты docker

- для создания и запуска контейнера воспользуемся командой **_docker run_**

    `docker run hello-world`

    ![docker run hello-world](source/docker_run_hello-world.png)

    Как видно на скриншоте, после создания и запуска контейнера, он выполнил приветствие и завершил свою работу.

- теперь создадим и запустим другой контейнер и подключимся к нему

    `docker run -it alpine`

    ![docker run alpine](source/docker_run_alpine.png)

    После создания и запуска контейнера он не завершился, а остался работать, т.к. мы подключены к терминалу в интерактивном режиме, и можем выполнять различные команды(например **cat**)

- после выхода из консоли контейнера, он остановится, т.к. основной процесс в нашем случае оболочка **sh** завершил свою работу.

    ![docker alpine exit](source/docker_exit_alpine.png)

    Так же на скриншоте можно увидеть, что при каждом запуске **_docker run_** создаётся и запускается новый контейнер.

- в случае если мы хотим запустить уже существующий контейнер, то можно воспользоваться командой **_docker start_**, передав в качестве аргумента имя контейнера или его ID.

    ![docker start alpine](source/docker_start_alpine.png)

- для подключения к запущенному контейнеру можно воспользоваться командой **_docker exec_** и передав в качестве аргументов имя или ID контейнера, и команду которую мы хотим запустить, в нашем случае оболочку **_sh_**

    `docker exec -it b7a940df2b94 sh`

    ![docker exec alpine](source/docker_exec_alpine.png)

- для остановки контейнера воспользуемся командой **_docker stop_**, передав в качестве аргумента имя или ID контейнера.

    `docker stop b7a940df2b9`

    ![docker stop alpine](source/docker_stop_alpine.png)

- для удаления контейнеров воспользуемся командой **_docker rm_**, передав в качестве аргумента имя или ID контейнера.

    `docker rm b7a940df2b94`

    ![docker rm alpine](source/docker_rm_alpine.png)

    В качестве аргументов команде **_docker rm_**, можно передать список элементов, например чтобы удалить все существующие контейнеры, можно воспользоваться следующей командой

    `docker rm $(docker ps -aq)`

    ![docker rm all](source/docker_rm_all.png)

- для получения образа из репозитория docker, нужно выполнить команду **_docker pull_**, передав в качестве аргумента имя образа

    `docker pull nginx`

    ![docker pull nginx](source/docker_pull_nginx.png)

    Для просмотра загруженных образов можно использовать команду **_docker images_**


- для удаления образа можно использовать команду **_docker rmi_**, передав в качестве аргумента имя или ID образа

    `docker rmi nginx`

    ![docker rmi nginx](source/docker_rmi_nginx.png)

    **_docker rmi_** может принимать в качестве аргументов список ID, и для удаления всех образов можно воспользоваться следующей командой

    `docker rmi $(docker images -q)`

    ![docker rmi all](source/docker_rmi_all.png)

Здесь были рассмотрены далеко не все команды и параметры, с которыми может работать утилита **docker**, подробную информацию можно получить в документации, используя параметр **--help** вместе с интересующей утилитой, например:

```
docker --help
docker run --help
docker exec --help
docker pull --help
```

### Хранение данных в контейнерах docker

При создании контейнера, ему выделяется своя собственная файловая система, которая будет хранить информацию во время существования контейнера, а при его удалении будет стёрта.
Для того чтобы избежать потери данных предусмотрен механизм монтирования данных внутрь файловой системы контейнера.

- мы можем указать для монтирования любую существующую директорию, например, создадим директорию _testfolder_ и укажем её в качестве аргумента при создании контейнера

    `docker run -it -v ./testfolder:/folder_inside alpine`

    ![docker volume ls](source/docker_volume_ls.png)

    Как видно на скриншоте, в файловой системе контейнера появилась новая директория.

- теперь мы можем работать с этой директорией изнутри контейнера, например создадим там файл и проверим, что мы можем его увидеть из хостовой системы

    ![docker ls and cat testfile](source/docker_ls_and_cat_testfile.png)

    В случае указания существующей точки монтирования внутри контейнера, эта директория будет заменена внешней.

    ![docker ls srv mountpoint](source/docker_ls_srv_mountpoint.png)

## Урок 4. Dockerfile и слои

### Синтаксис Dockerfile

Сам Dockerfile это набор инструкций которые будут выполнены для сборки образа.

- создадим файл с именем **_Dockerfile_**

    `touch Dockerfile`

- начнем c добавления базового образа с помощью инструкции **FROM**

    `FROM alpine`
    
    Если указать базовый образ без тега, будет использован тег **_latest_**, так же можно указать желаемую версию, например **_alpine:3.18_**

- инструкция **RUN** позволяет выполнить команду внутри контейнера, например, установить необходимые пакеты:

    `RUN apk update && apk --no-cache install bash`

    Т.к. каждая инструкция **RUN** добавляет слой изменений в образ, можно объединять команды с помощью **&&** для облегчения итогового образа.

- инструкция **WORKDIR** задаёт директорию выполнения команд.

    `WORKDIR /app`

    Эту инструкцию можно использовать несколько раз, если нужно выполнить команды в определённой директории.

- инструкция **COPY** позволяет скопировать данные из внешней файловой системы, нужно учесть что при сборке образа докер демон получит информацию только о текущем каталоге и подкаталогах, поднятся на уровень выше он не может. Т.е. нельзя указать в параметрах **COPY** путь **"../src"**.

    `COPY . /app`

    Эта команда скопирует все содержимое текущего каталога в папку **/app** внутри контейнера.
    На этом этапе следует учитывать, что докер демону передаётся вся информация из текущей директории, и если там есть данные не нужные для сборки контейнера, их можно перечислить в файле **.dockerignore**

- инструкция **CMD** позволяет задать выполнение команды, после создания контейнера на основе нашего образа

    `CMD ["bash"]`

    Так же можно задавать несколько аргументов, перечислив их через запятую.


- инструкция **ENTRYPOINT** позволяет задать команду, которая будет являтся точков входа, и которой можно будет передать внешние аргументы при запуске.

    `ENTRYPOINT ["bash"]`

Эте далеко не полный список инструкций, но его должно хватить для начального знакомства с построением образов. Более подробную информацию лучше получить в официальной документации (https://docs.docker.com/engine/reference/builder/).

### Добавление Dockerfile к существующему проекту

В качестве примера я буду использовать следующий проект:
https://github.com/RomanAnnenkov/GB_oop_calculator.git

- т.к следует ограничить видимость проекта при сборке, стоит начать с создания **.dockerignore** файла

    `touch .dockerignore`

    И добавления в него элементов которые будут исключены из зоны видимости docker

    ```
    **/.idea/
    *.md
    *.log
    **/target/
    **/logs/
    ```
- создаем Dockerfile, расположив его в корне проекта

    `touch Dockerfile`

    Чтобы исключить влияние внешней среды на сборку, и вместе с этим получить легковесный контейнер без лишних пакетов, разделим файл на два логических этапа:

    - создание образа сборщика, в который будут добавлены пакеты для компиляции приложения, и выполнения сборки проекта там

        `FROM alpine:3.18 AS builder`

        Задаем базовый образ, и через **AS** задаем имя, по которму сможем потом к нему обратиться 

        `RUN apk update && apk --no-cache add openjdk17-jdk=17.0.8_p7-r0 maven=3.9.2-r0`

        Обновляем списки пакетов, и устанавливаем необходимые для сборки с указанием версий.

        `COPY . .`

        Копируем файлы проекта в текущую директорию образа. 

        `RUN mvn package`

        Запускаем сборку проекта в текущей директории.

    - создание образа со средой выполнения, и копирование туда скомпилированного кода из сборщика

        `FROM alpine:3.18`

        Задаем базовый образ, для построения контейнера.

        `RUN apk update && apk --no-cache add openjdk17-jre=17.0.8_p7-r0`

        Обновляем и устанавливаем пакеты для исполнения приложения.

        `COPY --from=builder /target/*jar-with-dependencies.jar /app/app.jar`

        Копируем скомпилированный файл в папку внутри образа.

        `WORKDIR /app`

        Задаем рабочую директорию для выполения команд.

        `RUN mkdir logs && chown 1000:1000 logs`

        Создаем папку **logs** и переназначаем права, для возможности работы от непривилегированного пользователя.

        `CMD ["java", "-jar", "app.jar"]`

        Задаем команду для выполнения приложения в контейнере

- запускаем сборку образа

    `docker build -t calculator .`

- после завершения сборки запускаем контейнер с приложением

    `docker run -i -u 1000 -v ./logs:/app/logs calculator`

    В команде запуска задаём параметр **"-i"** т.к. приложение консольное и для работы понадобится интерактивный режим, задаем пользователя, для работы в непривелигированном режиме параметром **"-u 1000"**, так же в приложении используется логирование в файл, поэтому добавим монтирование директории **"-v ./logs:/app/logs calculator"**

    ![docker run app](source/docker_run_app.png)

## Урок 5. Docker Compose и Docker Swarm

### Запускаем сервисы используя _docker-compose_
Для работы **_docker-compose_** нужен файл с инструкция в формате **yml**, в нашем случае это будет **docker-compose.yml**
(**docker-compose.yml** - это значение по умолчанию для файла конфигурации, в случае если мы не укажем имя файла при запуске утилиты, будет задействоваи файл с таким именем)

- создадим файл **docker-compose.yml**

    `touch docker-compose.yml`

- добавим содержимое учитывая синтаксис **yml**
    ```
    version: '3'

    services:
      db:
        image: mariadb
        restart: always
        environment:
          MARIADB_ROOT_PASSWORD: password
        volumes:
          - ./db:/var/lib/mysql

      phpmyadmin:
        image: phpmyadmin
        restart: always
        ports:
          - 8080:80
    ```
- запустим контейнеры описанные в конфигурационном файле

    `docker-compose up -d`

    ![docker-compose up -d, ps](source/docker_compose_up_ps.png)


    Оба контейнера успешно запустились, проверить работоспособность можно обратившись на порт 8080.

- удалим контейнеры описанные в конфигурационном файле

    `docker-compose down`

    Эта команда остановит, а затем удалит контейнеры.

### Подготовка и запуск docker swarm кластера из 3-х виртуальных машин
Создадим 3 виртуальные машины объединённые одним сетевым пространством, и установим на них докер.
Все действия описанные ниже проиходят на ОС ubuntu 22.04, в случае если у вас другой дистрибутив или тип ОС, нужно будет корректировать этапы установки.

- для избежания ручной настройки воспользуемся утилитой **_vagrant_**, которая по иструкции описанной в файле(**Vagrantfile**) создаст виртуальные машины, и установит **_docker_**.

    - установим **_vagrant_** из репозитория

        `sudo apt install vagrant`

    - создадим файл **Vagrantfile**

        `touch Vagrantfile`

    - добавим содержимое учитывая синтаксис, и пользуясь документацией

        ```
        #
        $setup_docker = <<SCRIPT
        DEBIAN_FRONTEND=noninteractive
        for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
        # Add Docker's official GPG key:
        sudo apt-get update
        sudo apt-get install -y ca-certificates curl gnupg
        sudo install -m 0755 -d /etc/apt/keyrings
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
        sudo chmod a+r /etc/apt/keyrings/docker.gpg

        # Add the repository to Apt sources:
        echo \
          "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
          "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
          sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
        sudo apt-get update

        sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
        SCRIPT

        nodes = 3

        instances = []

        (1..nodes).each do |n| 
          instances.push({:name => "node#{n}", :ip => "192.168.56.#{n+10}"})
        end

        File.open("./hosts", 'w') { |file| 
          instances.each do |i|
            file.write("#{i[:ip]} #{i[:name]} #{i[:name]}\n")
          end
        }

        Vagrant.configure("2") do |config|
          config.vm.provider "virtualbox" do |v|
            v.memory = 512
            v.cpus = 1
          end

          instances.each do |instance|
            config.vm.define instance[:name] do |i|
              i.vm.box = "ubuntu_focal"
              i.vm.hostname = instance[:name]
              i.vm.network "private_network", ip: "#{instance[:ip]}"
              i.vm.provision "shell", inline: $setup_docker
              if File.file?("./hosts") 
                i.vm.provision "file", source: "hosts", destination: "/tmp/hosts"
                i.vm.provision "shell", inline: "cat /tmp/hosts >> /etc/hosts", privileged: true
              end
            end
          end
        end
        ```
    - для облегчения настройки системы воспользуемся готовым **box**

        `wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64-vagrant.box`

    - после загрузки добавим его в **_vagrant_**

        `vagrant box add ubuntu_focal focal-server-cloudimg-amd64-vagrant.box`

- для создания и запуска виртуальных машин, выполним в директории с **Vagrantfile** команду 

    `vagrant up`

    После завершения работы **_vagrant up_** наши виртальные машины готовы к работе

- подключимся ssh к первой ноде и создадим кластер
    
    `vagrant ssh node1`

    `sudo docker swarm init --advertise-addr 192.168.56.11`

    Так как у виртуальной машины несколько сетевых интерфейсов, необходимо указать интерфейс через который будет проходить взаимодействие между серверами кластера.

    В ответ на команду инициализации, будет сгенерирована команда, для присоединения к кластеру в качестве **worker**

- Для генерации ключа для подключения **manager** воспользуемся следующей командой на **node1**

    `sudo docker swarm join-token manager`

- подключимся по ssh на **node2** и **node3** и выполним команду подключения к кластеру с ролью **manager**

    ![docker node ls](source/docker_swarm_node_ls.png)

Таким образом у нас получился docker-swarm кластер из 3-х нод.

### Развёртывание сервисов в кластер

Для публикации сервисов, нам понадобится **yml** файл с инструкциями, которые мы будем передавать при развёртывании сервисов.

- Для разграничения запуска контейнеров по серверам, добавим **label** для каждой ноды

    `docker node update --label-add MARK=dev node1`

    `docker node update --label-add MARK=prod node2`

    `docker node update --label-add MARK=lab node3`

    ![docker swarm labels](source/docker_swarm_labels.png)

    Так как все ноды имеют статус **manager**, вносить изменения можно с любой ноды.

- Создадим 3 файла для разделения окружений

    `touch docker-compose.dev.yml docker-compose.prod.yml docker-compose.lab.yml`

- добавим содержимое в файлы на основе **docker-compose.yml**

    ```
    version: '3'

    services:
      db:
        image: mariadb
        environment:
          MARIADB_ROOT_PASSWORD: passworddev
        deploy:
          placement:
            constraints:
              - "node.labels.MARK==dev"

      phpmyadmin:
        image: phpmyadmin
        ports:
          - 8080:80
        deploy:
          placement:
            constraints:
              - "node.labels.MARK==dev"

    ```
    

    ```
    version: '3'

    services:
      db:
        image: mariadb
        environment:
          MARIADB_ROOT_PASSWORD: passwordprod
        deploy:
          placement:
            constraints:
              - "node.labels.MARK==prod"

      phpmyadmin:
        image: phpmyadmin
        ports:
          - 80:80
        deploy:
          placement:
            constraints:
              - "node.labels.MARK==prod"

    ```

    ```
    version: '3'

    services:
      db:
        image: mariadb
        environment:
          MARIADB_ROOT_PASSWORD: passwordlab
        deploy:
          placement:
            constraints:
              - "node.labels.MARK==lab"

      phpmyadmin:
        image: phpmyadmin
        ports:
          - 8081:80
        deploy:
          placement:
            constraints:
              - "node.labels.MARK==lab"

    ```

- развернем сервисы каждый в своём окружении

    `docker stack deploy -c docker-compose.lab.yml lab`

    `docker stack deploy -c docker-compose.prod.yml prod`

    `docker stack deploy -c docker-compose.dev.yml dev`

    ![docker swarm deploy services](source/docker_swarm_deploy_services.png)

    Сервисы развёрнуты, мы можем наблюдать за состоянием с любой ноды.
