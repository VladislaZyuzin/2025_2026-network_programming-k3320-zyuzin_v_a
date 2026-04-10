University: [ITMO University](https://itmo.ru/ru/)<br />
Faculty: [FICT](https://fict.itmo.ru)<br />
Course: [Network programming](https://github.com/itmo-ict-faculty/network-programming)<br /> 
Year: 2025/2026<br />
Group: K3320<br />
Author: Zyuzin Vladislav Alexandrowich<br />
Lab: Lab2<br />
Date of create: 10.04.2026<br />
Date of finished: ---<br />

# Задание

В данной лабораторной работе вы на практике ознакомитесь с системой управления конфигурацией Ansible, использующаяся для автоматизации настройки и развертывания программного обеспечения.

Цель работы: С помощью Ansible настроить несколько сетевых устройств и собрать информацию о них. Правильно собрать файл Inventory.

Ход работы:

1. Установить второй CHR на своем ПК.
2. Организовать второй OVPN Client на втором CHR.
3. Используя Ansible, настроить сразу на 2-х CHR:
  - логин/пароль;
  - NTP Client;
  - OSPF с указанием Router ID;
4. Собрать данные по OSPF топологии и полный конфиг устройства.

# Ход работы 

## Конфигурация 2-й роутера

Я склонировал в virtualbox старый микрот и создал новый, для этого нужно нажать на вот эту кнопку: 

<img width="376" height="534" alt="image" src="https://github.com/user-attachments/assets/624ca0a7-f361-466f-9101-626b3de73172" />

Так же я заново арендовал VPS, но уже с белым айпишником из России, чтобы товарищ Майор не стопорил трафик. Настроил второй микрот по аналогии с первым. Так, как у меня по какой то причине не коннектился winbox, хотя он видел айпишники, то я пострадал немного и занём ручками ключь на вм, потом после подключения я активировал ssh^
```rsc
/ip service enable ssh
```

После этого - можно поключаться к микроту по ssh и ловить дзен

Вот вывод `sudo wg show`:

<img width="534" height="328" alt="image" src="https://github.com/user-attachments/assets/9a2bf7ee-5956-4773-9c3f-dd3e83c2c7e5" />

После этого - нужно занести правило в iptables: 

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward

iptables -A FORWARD -i wg0 -j ACCEPT
iptables -A FORWARD -o wg0 -j ACCEPT
```

Форвардинг и правила в iptables дадут возможность пинговать микроты друг с другом

<img width="992" height="651" alt="image" src="https://github.com/user-attachments/assets/9ef9f74b-6f05-40b2-a192-e85b74c356c9" />

<img width="856" height="480" alt="image" src="https://github.com/user-attachments/assets/dd95b09b-f754-4af9-a607-fc4f4dcaacf6" />
