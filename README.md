# VPN_Wireguard_docker
Настройка своего VPN сервера

Поочередные команды:

Установка Docker

После чего установим docker. Можно последовать инструкциям по установке из документации, которые в случае с ubuntu говорят установить в систему репозиторий докера. Но в стандартных репозиториях ubuntu также можно найти не самую свежую, но подходящую версию. Ее и установим

apt install docker.io

и docker-compose

apt install docker-compose

Подготовка конфигов
Директория конфигов

Создадим, например в домашнем каталоге, директорию ~/wireguard, где будем хранить конфигурационные файлы, и сразу внутри нее директорию ~/wireguard/config, которую будем монтировать внутрь контейнера

mkdir -p ~/wireguard/config

docker-compose.yml

Далее создадим файл ~/wireguard/docker-compose.yml, в который скопируем содержимое из документации используемого docker образа

vim ~/wireguard/docker-compose.yml

В скопированном содержимом внесем некоторые изменения:

    Переменной окружения SERVERURL вместо текущего wireguard.domain.com нужно присвоить значение auto, или ip адреса вашего виртуального сервера.

    Переменной окружения PEERS нужно присвоить достаточное количество клиентов, которыми вы планируете пользоваться VPN. К примеру, если мы планируем подключаться с ноутбука, 2-х смартфонов и роутера (4 устройства), то установим PEERS=4

    Монтируемый каталог /path/to/appdata/config из секции volume необходимо заменить на ранее созданную директорию ~/wireguard/config

Итоговый файл будет выглядеть так:

---
version: "2.1"
services:
  wireguard:
    image: lscr.io/linuxserver/wireguard
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
      - SERVERURL=auto #optional
      - SERVERPORT=51820 #optional
      - PEERS=4 #optional
      - PEERDNS=auto #optional
      - INTERNAL_SUBNET=10.13.13.0 #optional
      - ALLOWEDIPS=0.0.0.0/0 #optional
    volumes:
      - ~/wireguard/config:/config
      - /lib/modules:/lib/modules
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped

На этом подготовка конфигурации окончена.
Запуск

Создадим и запустим контейнер

cd ~/wireguard
docker-compose up -d

Дождемся пока скачается образ и запустится контейнер, после проверим, что он запущен командой docker ps. При успехе должны увидеть в выводе что - то вроде

CONTAINER ID        IMAGE                   COMMAND             CREATED             STATUS              PORTS                      NAMES
97d91a1ffa87        linuxserver/wireguard   "/init"             5 minutes ago       Up 2 minutes        0.0.0.0:51820->51820/udp   wireguard

Если все так, то на этом настройка сервера закончена, и можно уже начать настройку клиентов.


Чтобы вывести на экран qr код добавления туннеля, на сервере исполняем команду:

docker exec -it wireguard /app/show-peer 1

Где 1 - порядковый номер PEERS, количество которых мы ранее задали в docker-compose.yml.

Конфиги подключения
Настройка клиента на любой платформе по сути сводится к тому, чтобы импортировать на клиент уже готовый файл конфигурации с сервера.
На сервере выведем содержимое монтируемой внутрь docker контейнера директории ~/wireguard/config

ls -la ~/wireguard/config

В этой директории наблюдаем файлы и директории жизнедеятельности контейнера. Нас интересуют директории peer1, peer2, peerN, где N - номер пира, как вы уже поняли.

Внутри каждой директории под каждого пира лежат заранее сгенерированные при старте контейнера файлы, а именно:

    приватный и публичный ключ (privatekey-peer1, publickey-peer1)
    peer1.png - изображение с qr кодом подключения
    peer1.conf - целевой конфигурационный файл, который требуется на клиенте
