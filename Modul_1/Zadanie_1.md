# Модуль 1. Настройка сетевой инфраструктуры

---

## Содержание

1. [Задание 1. Базовая настройка устройств](#1-задание-1-базовая-настройка-устройств)  
    1.1. [Имена устройств](#11-имена-устройств)  
    1.2. [Схема адресации IPv4](#12-схема-адресации-ipv4)  
    1.3. [Конфигурация сетевых интерфейсов](#13-конфигурация-сетевых-интерфейсов)  
    1.4. [Настройка VLAN на виртуальных машинах](#14-настройка-vlan-на-виртуальных-машинах)  

---

## 1. Задание 1. Базовая настройка устройств

### 1.1. Имена устройств

Выполнена настройка FQDN (полных доменных имён) согласно топологии.

**ISP**

```
hostnamectl set-hostname isp.au-team.irpo
```

**HQ-RTR**

```
hostnamectl set-hostname hq-rtr.au-team.irpo
```

**BR-RTR**

```
hostnamectl set-hostname br-rtr.au-team.irpo
```

**HQ-CLI**

```
hostnamectl set-hostname hq-cli.au-team.irpo
```

**HQ-SRV**

```
hostnamectl hostname hq-sru.au-team.irpo
```

**BR-SRV**

```
hostnamectl hostname br-sru.au-team.irpo
```

---

### 1.2. Схема адресации IPv4

Требования к подсетям и их реализация.

| Подсеть                 | VLAN | Требование                      | Маска    | Адресов |
|-------------------------|------|---------------------------------|----------|---------|
| В сторону HQ-SRV        | 100  | Не более 32 адресов             | /27      | 32      |
| В сторону HQ-CLI        | 200  | Не менее 16 адресов             | /28      | 16      |
| Сеть управления         | 999  | Не более 8 адресов              | /29      | 8       |
| В сторону BR-SRV        | —    | Не более 16 адресов             | /28      | 16      |

Сводная таблица назначенных IP-адресов (маршрутизаторы):

| Устройство | Интерфейс | Назначение        | IP-адрес       | Маска |
|------------|-----------|-------------------|----------------|-------|
| HQ-RTR     | eth0      | Стык с ISP        | 172.16.1.2     | /28   |
| HQ-RTR     | eth1.100  | Подсеть HQ-SRV    | 192.168.1.1    | /27   |
| HQ-RTR     | eth1.200  | Подсеть HQ-CLI    | 192.168.2.1    | /28   |
| HQ-RTR     | eth1.999  | Сеть управления   | 192.168.99.1   | /29   |
| BR-RTR     | eth0      | Стык с ISP        | 172.16.2.2     | /28   |
| BR-RTR     | eth1      | Подсеть BR-SRV    | 192.168.3.1    | /28   |

Сводная таблица назначенных IP-адресов (серверы и клиенты):

| Устройство | IP-адрес       | Шлюз          |
|------------|----------------|---------------|
| HQ-SRV     | 192.168.1.2/27 | 192.168.1.1   |
| BR-SRV     | 192.168.3.2/28 | 192.168.3.1   |

---

### 1.3. Конфигурация сетевых интерфейсов

#### 1.3.1. HQ-RTR

Файл: `/etc/network/interfaces`

```
auto eth0
iface eth0 inet static
    address 172.16.1.2/28
    gateway 172.16.1.1

auto eth1.100
iface eth1.100 inet static
    address 192.168.1.1/27
    vlan_raw_device eth1

auto eth1.200
iface eth1.200 inet static
    address 192.168.2.1/28
    vlan_raw_device eth1

auto eth1.999
iface eth1.999 inet static
    address 192.168.99.1/29
    vlan_raw_device eth1
```

Назначение интерфейсов:

- eth0 — в сторону офиса ISP
- eth1 — в сторону офиса HQ

Применение:

```
systemctl restart networking
```

#### 1.3.2. BR-RTR

Файл: `/etc/network/interfaces`

```
auto eth0
iface eth0 inet static
    address 172.16.2.2/28
    gateway 172.16.2.1

auto eth1
iface eth1 inet static
    address 192.168.3.1/28
```

Назначение интерфейсов:

- eth0 — в сторону ISP
- eth1 — в сторону офиса BR

Применение:

```
systemctl restart networking
```

#### 1.3.3. HQ-SRV

Файл: `/etc/net/ifaces/ens18/options`

```
BOOTPROTO=static
TYPE=eth
DISABLED=no
```

Файл: `/etc/net/ifaces/ens18/ipv4address`

```
192.168.1.2/27
```

Файл: `/etc/net/ifaces/ens18/ipv4route`

```
default via 192.168.1.1
```

Дополнительно: настроен VLAN-тег 100 на интерфейсе виртуальной машины в Proxmox VE.

#### 1.3.4. BR-SRV

Файл: `/etc/net/ifaces/ens18/options`

```
BOOTPROTO=static
TYPE=eth
DISABLED=no
```

Файл: `/etc/net/ifaces/ens18/ipv4address`

```
192.168.3.2/28
```

Файл: `/etc/net/ifaces/ens18/ipv4route`

```
default via 192.168.3.1
```

---

### 1.4. Настройка VLAN на виртуальных машинах

Параметры виртуальных машин в среде Proxmox VE.

**Структура датацентра:**

```
Datacenter
  pve
    100 (ISP)
    101 (HQ-RTR)
    102 (BR-RTR)
    103 (HQ-SRV)
    104 (HQ-CLI)
    105 (BR-SRV)
    106 (deb-template)
    localnetwork (pve)
    local (pve)
    local-lvm (pve)
    vms (pve)
```

**Назначение VLAN-тегов:**

| Виртуальная машина | Bridge    | VLAN Tag |
|--------------------|-----------|----------|
| HQ-SRV (103)       | HQ_SW     | 100      |
| HQ-CLI (104)       | HQ_SW     | 200      |

Конфигурация сетевых устройств:

- **HQ-SRV (net0):** `virtio=BC:24:11:A8:C9:54, bridge=HQ_SW`
- **HQ-CLI (net0):** `virtio=BC:24:11:F3:87:3A, bridge=HQ_SW, firewall=1, tag=200`
