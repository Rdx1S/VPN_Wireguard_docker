# VPN_Wireguard_docker
Настройка своего VPN сервера

Поочередные команды:
```
ssh root@ip-адрес-сервера
```
Обновление системы
Сначала необходимо обновить кеш репозиториев и систему.
```
apt update && apt upgrade -y
```
# Установка Docker

После чего установим docker. Можно последовать инструкциям по установке из документации, которые в случае с ubuntu говорят установить в систему репозиторий докера. Но в стандартных репозиториях ubuntu также можно найти не самую свежую, но подходящую версию. Ее и установим
```
apt install docker.io
```
и docker-compose
```
apt install docker-compose
```
# Подготовка конфигов

Директория конфигов
Создадим, например в домашнем каталоге, директорию ~/wireguard, где будем хранить конфигурационные файлы, и сразу внутри нее директорию ~/wireguard/config, которую будем монтировать внутрь контейнера
```
mkdir -p ~/wireguard/config
```
Далее создадим файл ~/wireguard/docker-compose.yml, в который скопируем содержимое из документации используемого docker образа
```
nano ~/wireguard/docker-compose.yml
```
В скопированном содержимом внесем некоторые изменения:

    Переменной окружения SERVERURL вместо текущего wireguard.domain.com нужно присвоить значение auto, или ip адреса вашего виртуального сервера.

    Переменной окружения PEERS нужно присвоить достаточное количество клиентов, которыми вы планируете пользоваться VPN. К примеру, если мы планируем подключаться с ноутбука, 2-х смартфонов и роутера (4 устройства), то установим PEERS=4

    Монтируемый каталог /path/to/appdata/config из секции volume необходимо заменить на ранее созданную директорию ~/wireguard/config

Итоговый файл будет выглядеть так:
```
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
 ```
 На этом подготовка конфигурации окончена.
 
# Запуск

Создадим и запустим контейнер
```
cd ~/wireguard
docker-compose up -d
```
Дождемся пока скачается образ и запустится контейнер, после проверим, что он запущен командой docker ps. При успехе должны увидеть в выводе что - то вроде
```
CONTAINER ID        IMAGE                   COMMAND             CREATED             STATUS              PORTS                      NAMES
97d91a1ffa87        linuxserver/wireguard   "/init"             5 minutes ago       Up 2 minutes        0.0.0.0:51820->51820/udp   wireguard
```
Если все так, то на этом настройка сервера закончена, и можно уже начать настройку клиентов.

Чтобы вывести на экран qr код добавления туннеля, на сервере исполняем команду:
```
docker exec -it wireguard /app/show-peer 1
```
Где 1 - порядковый номер PEERS, количество которых мы ранее задали в docker-compose.yml.
Рекомендуется каждый клиент подключать под отдельным peer.

# Конфиги подключения

Настройка клиента на любой платформе по сути сводится к тому, чтобы импортировать на клиент уже готовый файл конфигурации с сервера.

На сервере выведем содержимое монтируемой внутрь docker контейнера директории ~/wireguard/config
```
ls -la ~/wireguard/config
```
В этой директории наблюдаем файлы и директории жизнедеятельности контейнера. Нас интересуют директории peer1, peer2, peerN, где N - номер пира, как вы уже поняли.

Внутри каждой директории под каждого пира лежат заранее сгенерированные при старте контейнера файлы, а именно:

    приватный и публичный ключ (privatekey-peer1, publickey-peer1)
    peer1.png - изображение с qr кодом подключения
    peer1.conf - целевой конфигурационный файл, который требуется на клиенте


# GNU/Linux

Чтобы установить wireguard клиент в свой linux дистрибутив, необходимо установить пакет wireguard-tools из репозиториев:
Установка клиента
Arch linux
```
sudo pacman -S wireguard-tools
```
Debian/Ubuntu
```
sudo apt install wireguard
```
Fedra
```
sudo dnf install wireguard-tools
```
Инструкции по другим дистрибутивам можно найти на странице установки на официальном сайте.
Конфиг

После установки нам необходимо на клиентской машине создать конфигурационный файл подключения /etc/wireguard/wg0.conf с содержимым файла ~/wireguard/config/peer1/peer1.conf с сервера.

Для этого можно например вывести содержимое файла peer1.conf в консоль командой cat ~/wireguard/config/peer1/peer1.conf, и скопипастить содержимое в целевой файл на клиенте.

Но вместо этого можно скачать файл при помощи утилиты scp с сервера прямо в целевой файл следующей единственной командой:
```
sudo scp root@ip-адрес-сервера:~/wireguard/config/peer1/peer1.conf /etc/wireguard/wg0.conf
```
    sudo при запуске потребуется для того, чтобы записать содержимое в файл, находящийся вне домашней директории вашего пользователя ОС (/etc).

    первым параметром после указания имени утилиты scp указывается строка подключения к серверу ssh. Затем двоеточие, после которого путь к скачиваемому файлу на сервере.

    Вторым параметром указывается путь к файлу, который будет создан на локальной машине.

Запуск

Все готово, осталось лишь запустить клиент следующей командой:
```
wg-quick up wg0
```
Проверим, что все работает:
```
curl ifconfig.me
195.255.255.255
```
Если в выводе ip адрес вашего сервера, то все получилось!
Остановка клиента
```
wg-quick down wg0
```


# Raspberry client
Чтобы установить WireGuard на Raspberry Pi в качестве клиента, выполните следующие шаги:

Установите операционную систему на Raspberry Pi. Можно использовать Raspbian, Ubuntu или другую совместимую с ARM операционную систему.
Установите пакет WireGuard на Raspberry Pi. Для этого откройте терминал на Raspberry Pi и выполните следующую команду:
```
sudo apt update
sudo apt install wireguard
```
Эта команда установит все необходимые пакеты для работы WireGuard на Raspberry Pi.
Создайте ключи для клиента. На Raspberry Pi выполните следующую команду:
```
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
```
Эта команда создаст пару ключей: приватный ключ (privatekey) и публичный ключ (publickey).

Настройте конфигурационный файл WireGuard на Raspberry Pi. Создайте новый файл с именем wg0.conf и откройте его для редактирования. Вставьте следующий текст:
```
[Interface]
PrivateKey = <вставьте приватный ключ вашего клиента>
Address = 10.0.0.2/24

[Peer]
PublicKey = <вставьте публичный ключ сервера>
Endpoint = <IP-адрес сервера>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 21
```
В этом файле вы настраиваете интерфейс WireGuard (PrivateKey и Address) и определяете связь между Raspberry Pi и сервером (PublicKey, Endpoint и AllowedIPs). Параметр PersistentKeepalive гарантирует, что соединение останется активным даже при отсутствии активности.

Запустите WireGuard на Raspberry Pi. Выполните следующую команду, чтобы запустить WireGuard:
```
sudo wg-quick up wg0
```
Теперь ваш Raspberry Pi должен быть подключен к серверу WireGuard.

# Добавление подключения к VPN  в автозапуск

Чтобы автоматически запускать WireGuard при загрузке Raspberry Pi, необходимо добавить соответствующие команды в файл автозапуска.
Откройте терминал на Raspberry Pi и выполните следующую команду, чтобы открыть файл rc.local для редактирования:
'''
sudo nano /etc/rc.local
'''
Добавьте следующие команды в файл rc.local, перед строкой "exit 0":
'''
sudo wg-quick up wg0
'''
Сохраните изменения, нажав "Ctrl+O", а затем выйдите из редактора, нажав "Ctrl+X".
Перезагрузите Raspberry Pi, чтобы убедиться, что WireGuard запускается автоматически при загрузке.
Теперь при каждой загрузке Raspberry Pi WireGuard будет автоматически подключаться к серверу, используя конфигурационный файл wg0.conf.
