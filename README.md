# instagram-router-filter

## Использование OpenVPN для обхода блокировок
Решение ниже позволяет использовать VPN-соедиенение исключтельно для списка IP instagram/Facebook. При обращении к любым другим ресурсам будет использоваться привычное провайдерское соединение.

### Требования
* Прошивка с ipset (любые сборки Padavan, кроме nano),
* Работающее OpenVPN-соединение.

### Установка скриптов

* Скачайте список IP адресов facebook/instagram:
```
wget -O /opt/etc/rublock.ips https://raw.githubusercontent.com/SecOps-Institute/FacebookIPLists/master/facebook_ipv4_cidr_blocks.lst
wget -O /opt/etc/init.d/S10iptables https://raw.githubusercontent.com/diarworld/instagram-router-filter/main/S10iptables
```

Повторно запускать скачивание имеет смысл только для обновления списка заблокированных IP адресов, например, раз в месяц.


### Настройка прошивки

* В веб-интерфейсе роутера на странице `Customization > Scripts` отредактируйте поле `Run After Router Started`, раскоментировав две строчки:
```
modprobe ip_set_hash_ip
modprobe xt_set
```
* На странице `VPN Client` в поле `OpenVPN Extended Configuration` допишите следующую команду:
```
route-noexec
```
Теперь не будут применяться правила роутинга, которые будут переданы с OpenVPN-сервера. Убедитесь, что пункт `Route All Traffic through the VPN interface?` переключен в `No`.
* На той же странице приведите скрипт `Run the Script After Connected/Disconnected to VPN Server` к следующему виду:
```
#!/bin/sh

func_ipup()
{
    echo 0 > /proc/sys/net/ipv4/conf/$IFNAME/rp_filter
    ip route flush table 1
    ip rule del table 1
    ip rule add fwmark 1 table 1 priority 1000
    ip route add default via $route_vpn_gateway table 1
    return 0
}

func_ipdown()
{
   return 0
}

logger -t vpnc-script "$IFNAME $1"

case "$1" in
up)
  func_ipup
  ;;
down)
  func_ipdown
  ;;
esac
```

Перегрузите роутер для того, чтобы настройки вступили в силу.

### Примечание

Решение для обхода всех блокировок (не работает из-за слишком разроссшегося списка блокировок, на большеё части роуетров падает): https://github.com/DontBeAPadavan/rublock-via-vpn/wiki. Также изменился адрес api у rublacklist, необходимо выполнить следующее:
```
sed -i 's/blSource = "antizapret"/blSource = "rublacklist"/' /opt/bin/blupdate.lua
sed -i 's/http:\/\/reestr.rublacklist.net\/api\/current/https:\/\/reestr.rublacklist.net\/api\/v2\/current\/csv/' /opt/bin/blupdate.lua
```

