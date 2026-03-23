University: [ITMO University](https://itmo.ru/ru/)<br />
Faculty: [FICT](https://fict.itmo.ru)<br />
Course: [Network programming](https://github.com/itmo-ict-faculty/network-programming)<br /> 
Year: 2025/2026<br />
Group: K3320<br />
Author: Zyuzin Vladislav Alexandrovich<br />
Lab: Lab1<br />
Date of create: 20.03.2026<br />
Date of finished: 22.03.2026<br />

# Задание

Данная работа предусматривает обучение развертыванию виртуальных машин (VM) и системы контроля конфигураций Ansible а также организации собственных VPN серверов.

Цель работы: развертывание виртуальной машины на базе платформы Microsoft Azure с установленной системой контроля конфигураций Ansible и установка CHR в VirtualBox

Ход работы:

Вам необходимо развернуть виртуальную машину с помощью Microsoft Azure в режиме студенческой подписки.

Если не получается в Microsoft Azure, можете выбрать любого бесплатного облачного провайдера

В бесплатном режиме Microsoft Azure предлагает для виртуальных машин только Ubuntu 16.4, нам нужна Ubuntu 18.+ поэтому необходимо обновить операционную систему. Сделать это можно с помощью данных команд:

```
sudo apt update & sudo apt upgrade
sudo do-release-upgrade
```

Теперь необходимо установить python3 и Ansible:

```
sudo apt install python3-pip
ls -la /usr/bin/python3.6
sudo pip3 install ansible
ansible --version
```

Далее вам необходимо на вашем компьютере установить VirtualBox а на него CHR (RouterOS).


После этого вам необходимо создать свой Wireguard/OpenVPN/L2TP сервер для организации VPN туннеля между вашим сервером автоматизации где был установлена система контроля конфигураций Ansible и вашим локальным CHR.


После всех манипуляций вам необходимо будет поднять VPN туннель между вашим VPN сервером на Ubuntu 18 и VPN клиентом на RouterOS (CHR)

# Ход работы 

## Создание VPS и создание ВМ в virtualbox на микротике

Для начала - я создал vps на платформе с белым айпишником, после чего полез качать образ CHR (советую воспользоваться этой докой, тут всё станет +- понятно: https://getlabsdone.com/how-to-install-mikrotik-router-on-virtualbox/) для того, чтобы у меня запустился микротик я скачал с официального сайта [https://mikrotik.com/download/chr] VMDK диск. Только с ним у меня нормально завёлся микрот.

Так же искринне советую поставить [winbox](https://mikrotik.com/download/winbox), так как вводить ручками ключи без функции ctrl + C и ctrl + V вы вряд ли захотите, а тут хорошая гуи

Советую настроить сеть, как советуется в доке. Там всё предельно понятно

В конечном итоге всё будет выглядеть так: 

<img width="937" height="1015" alt="image" src="https://github.com/user-attachments/assets/8cb1bef4-91d1-45c6-b4dc-27e83903c9b1" />

При входе на роутер, вид будет следующим:

<img width="715" height="504" alt="image" src="https://github.com/user-attachments/assets/42ae57b6-090e-4304-b052-5dd2f5b50dac" />

## Настройка соединения

Вернёмся ненадолго к VPS. На неё я установил уже ансамбль, далее - я установил wireguard:
```bash
sudo apt update && sudo apt install wireguard wireguard-tools -y
```

Сгенерил ключи и положил их в папку `/etc/wireguard/keys`:
```bash
sudo mkdir /etc/wireguard/keys
wg genkey | sudo tee /etc/wireguard/keys/server_private.key | wg pubkey | sudo tee /etc/wireguard/keys/server_public.key
```

Создаём переменную с сохранением приватного ключа: 
```bash
SERVER_PRIVATE_KEY=$(sudo cat /etc/wireguard/keys/server_private.key)
```
Если вам всё рано на безопасность - то просто вводите её в конфиг ниже

Далее - я записал конфиг для работы VPN-соединения `/etc/wireguard/wg0.conf`:
```
[Interface]
Address = 10.10.0.1/24
ListenPort = 51820
PrivateKey = ${SERVER_PRIVATE_KEY}

[Peer]
PublicKey = <public_key_chr>
AllowedIPs = 10.10.0.2/32
```
на vps я разрешил форвардинг и открыл порт: 
```
sudo sysctl -w net.ipv4.ip_forward=1
sudo ufw allow 51820/udp
```

Теперь посмотрим на chr:

Пропишем следующее:
```rsc
/interface wireguard
add listen-port=51820 name=wireguard0
/interface wireguard print  # (нам понадобится публичный ключь)
/ip address
add address=10.10.0.2/32 interface=wireguard0
```

Вставляем публичный ключь в конфигурацию, которую мы написали выше в VPS. Там же обновляем и активируем сервис `wg-quick@wg0`:

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```
Возвращаемся к chr:

Пропишем туда данные (вводите это лучше через гуи, так как замучаетесь вводить публичный ключь wireguard-а с VPS):

```rsc
/interface wireguard peers
add allowed-address=10.10.0.1/24 interface=wireguard0 endpoint-address=IP_VPS endpoint-port=51820 public-key="ПУБЛИЧНЫЙ_КЛЮЧ_VPS"
```
Не звбываем добавить маршрут и настроить фаервол
```rsc
/ip route
add dst-address=10.10.0.0/24 gateway=wireguard0

/ip firewall filter
add action=accept chain=input dst-port=51820 protocol=udp src-address=IP_VPS
```

После чего пингуем VPS: 

<img width="739" height="407" alt="Снимок экрана 2026-03-22 205911" src="https://github.com/user-attachments/assets/fbbb76fd-e30a-48f6-a269-66f171c26d41" />

Если пинг проходит, то всё хорошо

Так же, если на самой VPS отображается откуда шёл пинг, то всё становится вообще отлично:

<img width="738" height="250" alt="image" src="https://github.com/user-attachments/assets/b355e8d4-4673-4030-9ad3-b21c23e0620f" />

У меня с VPS видно, что коннект есть и впс с микротом обменивались данными, причём я немного запалил свой айпи адрес, он питерский у микрота. Значит всё отработало штатно

# Заключение 

В ходе работы мною были получены навыки работы с chr (микротом), wireguard и ансамблем (мы по нему связывали впс с микротом из конфига). В итоге у меня получилось настроить впн-соединение между удалённым vps и микротром на virtualbox моего ноутбука
