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

После того, как мы проверили, что можем на них зайти с vps по ssh, то можно приступать к настройке ансамбля. Качнём нужные коллекции:

```bash
ansible-galaxy collection install community.routeros
ansible-galaxy collection install ansible.netcommon
```

После чего прописываем инвентори:

```yaml
[routers]
chr1 ansible_host=10.100.0.2
chr2 ansible_host=10.100.0.3

[routers:vars]
ansible_connection=network_cli
ansible_network_os=community.routeros.routeros
ansible_user=ansible
ansible_password=123
```

Для проверки выполним: 

<img width="1255" height="514" alt="image" src="https://github.com/user-attachments/assets/c3dc8c3f-2893-4199-b5fe-1120e1a81daa" />

После чего - пропишем плейбук:

```yaml
- name: Configure and Collect Data from MikroTik
  hosts: routers
  gather_facts: no

  tasks:
    # 1. Подготовка локальной среды
    - name: Ensure configs directory exists on host
      delegate_to: localhost
      file:
        path: "./configs"
        state: directory

    # 2. Настройка пользователя (Требование ТЗ: логин/пароль)
    - name: Create admin user
      community.routeros.command:
        commands:
          - /user add name=admin_vlad password=SuperSecretPassword123 group=full
      ignore_errors: yes

    # 3. Системные настройки
    - name: Set identity
      community.routeros.command:
        commands:
          - /system identity set name={{ inventory_hostname }}

    - name: Configure NTP Client
      community.routeros.command:
        commands:
          - /system ntp client set enabled=yes
          - /system ntp client servers add address=pool.ntp.org
      ignore_errors: yes

    # 4. Настройка OSPF (Синтаксис v7)
    - name: Create OSPF Instance
      community.routeros.command:
        commands:
          - /routing ospf instance add name=default-v2 router-id={{ ansible_host }} disabled=no
      ignore_errors: yes

    - name: Create OSPF Area
      community.routeros.command:
        commands:
          - /routing ospf area add name=backbone-v2 instance=default-v2 area-id=0.0.0.0 disabled=no
      ignore_errors: yes

    - name: Create OSPF Interface Template
      community.routeros.command:
        commands:
          - /routing ospf interface-template add networks=10.100.0.0/24 area=backbone-v2 type=ptp
      ignore_errors: yes

    # 5. Сбор фактов и выполнение команд для отчета
    - name: Gather facts
      community.routeros.facts:
        gather_subset:
          - default
      register: device_facts

    - name: Collect OSPF state and Full Config
      community.routeros.command:
        commands:
          - /routing ospf neighbor print
          - /export compact
      register: show_commands

    # 6. Сохранение результатов в файлы
    - name: Save configuration to local file
      delegate_to: localhost
      copy:
        content: "{{ show_commands.stdout[1] }}"
        dest: "./configs/{{ inventory_hostname }}_config.txt"

    - name: Save OSPF neighbors to local file
      delegate_to: localhost
      copy:
        content: "{{ show_commands.stdout[0] }}"
        dest: "./configs/{{ inventory_hostname }}_ospf.txt"
```

Корректеая работа




<img width="992" height="651" alt="image" src="https://github.com/user-attachments/assets/9ef9f74b-6f05-40b2-a192-e85b74c356c9" />

<img width="856" height="480" alt="image" src="https://github.com/user-attachments/assets/dd95b09b-f754-4af9-a607-fc4f4dcaacf6" />


