# 4. Конфигурирование DHCP. Ретрансляция сообщений DHCP

## Вводная информация
| Сеть        | Адрес       | Маска подсети   |
|------------ |-------------|-----------------|
| A (слева)   | 192.168.1.0 | 255.255.255.0   |
| B (справа)  | 192.168.2.0 | 255.255.255.0   |
| M (M1 ↔ M2) | 10.0.0.0    | 255.255.255.252 |

> [!WARNING]
> Сеть M использует маску бесклассовой адресации (255.255.255.252), для работы которой требуется включение RIP v2 на соответствующих маршрутизаторах.

### Топология
![topology](https://i.imgur.com/rasnj1N.png)

## Базовая конфигурация сети
Маршрутизация в сети будет осуществляться статически.

### Маршрутизатор M1
```yaml
int fa0/0
    ip address 192.168.1.1 255.255.255.0
    no shutdown
    exit

int s2/0
    ip address 10.0.0.1 255.255.255.252
    no shutdown
    exit

# Конфигурация статической маршрутизации
ip route 192.168.2.0 255.255.255.0 10.0.0.2
```

### Маршрутизатор M2
```yaml
int fa0/0
    ip address 192.168.2.1 255.255.255.0
    no shutdown
    exit

int s2/0
    ip address 10.0.0.2 255.255.255.252
    no shutdown
    exit

ip route 192.168.1.0 255.255.255.0 10.0.0.1
```

## Конфигурирование сервера DHCP

### Маршрутизатор M1
Машрутизатор M1 будет иметь запущенный DHCP-сервер.
```yaml
# Конфигурация пула DHCP для сети A
ip dhcp pool LAN-A
    network 192.168.1.0 255.255.255.0
    default-router 192.168.1.1        # указание шлюза по умолчанию
    dns-server 8.8.8.8                # указание адреса DNS-сервера
    exit

# Исключение адреса шлюза по умолчанию из пула
ip dhcp excl 192.168.1.1

# Конфигурация пула DHCP для сети B
ip dhcp pool LAN-B
    network 192.168.2.0 255.255.255.0
    default-router 192.168.2.1        # указание шлюза по умолчанию
    dns-server 8.8.8.8                # указание адреса DNS-сервера
    exit

# Исключение адреса шлюза по умолчанию из пула
ip dhcp excl 192.168.2.1

# Включение сервера DHCP
service dhcp
```

### Маршрутизатор M2
На машрутизаторе M2 требует настроить ретрансляцию сообщений DHCP, чтобы хосты сети B смогли получить необходимую DHCP конфигурацию от маршрутизатора M1.
```yaml
# Задание ретрансляции на интерфейсе сети B
int fa0/0
    ip helper-address 10.0.0.1
```

## Проверка
Все хосты всех сетей должны получить:
- адрес шлюза по умолчанию,
- IP-адрес,
- маску подсети,
- адрес DNS сервера.

Вывести статистику работы DHCP
```
show ip dhcp pool
```
```yaml
Pool LAN-A :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0 
 Total addresses                : 254
 Leased addresses               : 2
 Excluded addresses             : 2
 Pending event                  : none

 1 subnet is currently in the pool
 Current index        IP address range                    Leased/Excluded/Total
 192.168.1.1          192.168.1.1      - 192.168.1.254     2    / 2     / 254

Pool LAN-B :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0 
 Total addresses                : 254
 Leased addresses               : 1
 Excluded addresses             : 2
 Pending event                  : none

 1 subnet is currently in the pool
 Current index        IP address range                    Leased/Excluded/Total
 192.168.2.1          192.168.2.1      - 192.168.2.254     1    / 2     / 254
```