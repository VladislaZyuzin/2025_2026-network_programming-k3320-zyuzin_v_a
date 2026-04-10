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
- name: Lab2 Setup
  hosts: routers
  gather_facts: no

  tasks:
    - name: Prepare local directory
      delegate_to: localhost
      file:
        path: "./configs"
        state: directory

    - name: Connectivity and Firewall fixes
      community.routeros.command:
        commands:
          - /ip firewall filter add chain=input protocol=ospf action=accept place-before=0 comment="Allow OSPF"
          - /interface wireguard set [find] mtu=1300
      ignore_errors: yes

    - name: User setup
      community.routeros.command:
        commands:
          - /user add name=vlad_admin password=SafePassword2026! group=full
          - /system identity set name={{ inventory_hostname }}
          - /system ntp client set enabled=yes servers=pool.ntp.org
      ignore_errors: yes

    - name: OSPF v7 Setup
      community.routeros.command:
        commands:
          - /routing ospf instance add name=inst router-id={{ ansible_host }} disabled=no
          - /routing ospf area add name=backbonev2 instance=inst area-id=0.0.0.0
          - /routing ospf interface-template add area=backbonev2 networks=10.100.0.0/24 type=ptp
          # Прописываем соседа: для chr1 это .3, для chr2 это .2
          - "/routing ospf static-neighbor add address={{ '10.100.0.3' if inventory_hostname == 'chr1' else '10.100.0.2' }} instance=inst"
      ignore_errors: yes

    - name: Collect evidence
      community.routeros.command:
        commands:
          - /export compact
          - /routing ospf neighbor print detail
      register: chr_output

    - name: Save result files
      delegate_to: localhost
      copy:
        content: "{{ item.content }}"
        dest: "./configs/{{ inventory_hostname }}_{{ item.suffix }}"
      loop:
        - { content: "{{ chr_output.stdout[0] }}", suffix: "export.txt" }
        - { content: "NEIGHBORS:\n{{ chr_output.stdout[1] }}", suffix: "ospf.txt" }
```



Корректеая работа

<img width="992" height="651" alt="image" src="https://github.com/user-attachments/assets/9ef9f74b-6f05-40b2-a192-e85b74c356c9" />

<img width="856" height="480" alt="image" src="https://github.com/user-attachments/assets/dd95b09b-f754-4af9-a607-fc4f4dcaacf6" />

Так же - проверим ntp: 

<img width="514" height="439" alt="image" src="https://github.com/user-attachments/assets/e2acd68f-35a0-41c2-9898-611b7d24f1af" />

И юзера (я делал несколько версий плейбука, так что пользователь может отличаться): 

<img width="544" height="150" alt="image" src="https://github.com/user-attachments/assets/d54a17a4-5712-497c-b425-4c7f88bb6bb5" />

