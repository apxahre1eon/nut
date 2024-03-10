# nut для бюджетных Ippon
Устанавливаем NUT:
```
root@proxmox:~# apt update && apt install nut -y
```
Ищем наш UPS:
```
root@proxmox:~# lsusb
```
В ответ получаем что-то типа:
```
Bus 002 Device 002: ID 8087:8002 Intel Corp. 8 channel internal hub
Bus 002 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 001 Device 002: ID 8087:800a Intel Corp. Hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 002: ID 0665:5161 Cypress Semiconductor USB to Serial
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```
В данном случае, наш UPS: </br>
`Bus 003 Device 002: ID 0665:5161 Cypress Semiconductor USB to Serial` </br>

Для поиска подходящего драйвера можно использовать авто сканнер nut-scanner
```
root@proxmox:~# nut-scanner 
Scanning USB bus.
No start IP, skipping NUT bus (old connect method)
Scanning NUT bus (avahi method).
Failed to create client: Daemon not running
[nutdev1]
	driver = "nutdrv_qx"
	port = "auto"
	vendorid = "0665"
	productid = "5161"
	product = "USB to Serial"
	serial = "20100826"
	vendor = "INNO TECH"
	bus = "003"
```
В данном случае верно определивший наш UPS и сразу подобравший для него драйвер. Берём блок [nutdev1] и добавляем его в `/etc/nut/ups.conf`
```
[ippon]
        driver = "nutdrv_qx"
        port = "auto"
        vendorid = "0665"
        productid = "5161"
        product = "USB to Serial"
        serial = "20100826"
        vendor = "INNO TECH"
        bus = "003"
        desc = "Ippon Back Basic 650S Euro"
        override.battery.voltage.nominal = 12.00
        default.battery.voltage.high = 13.60
        default.battery.voltage.low = 10.40
```
Атрибуты: </br>
`[ippon]` Имя ИБП для NUT, с помощью которого и будем производить все манипуляции с ИБП </br>
`driver` Драйвер ИБП </br>
`port` порт, на котором висит UPS (для подключения через USB указываете значение "auto". Для snmp-ups: имя хоста SNMP агента. Для newhidups: значение "auto" для автоматического соединения с USB UPS </br>
Идентификатор устройства </br>
`vendorid` </br>
`productid` </br>
`product` Тип устройства. В данном случаем внутри ИБП уже установлен конвертор USB to Serial </br>
`serial` Серийный номер </br>
`vendor` Бренд изготовителя устройства </br>
`bus` канал/шина на котором расположен порт </br>
`desc` - для удобства добавляем имя и\или расположение ИБП </br>
Номинальное напряжение АКБ 12V: </br>
`override.battery.voltage.nominal = 12.00` </br>
Напряжение для 100% заряда: </br>
`default.battery.voltage.high = 13.60` </br>
Минимальное напряжение АКБ: </br>
`default.battery.voltage.low = 10.40` </br>

Чтобы сервер работал автономно, добавим в `/etc/nut/nut.conf` "MODE=standalone" </br>
```
root@proxmox:~# echo "MODE=standalone" > /etc/nut/nut.conf
```
