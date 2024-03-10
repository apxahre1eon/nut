# nut standalone для бюджетных Ippon 
Устанавливаем nut:
```
root@proxmox:~# apt update && apt install nut -y
```

Все конфигурационные файлы находятся в директории `/etc/ups` </br>
`ups.conf` - настройки nut для работы с ИБП (драйвер/порт/наименование) </br>
`upsd.conf` - настройка основного демона upsd Network UPS Tools </br>
`upsd.users` - контроль доступа к ИБП демону (профили пользователей) </br>
`upsmon.conf` - настройка текущего клиентского агента </br>
</br>
Ищем наш ИБП:
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
В данном случае, наш ИБП: </br>
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
В данном случае верно определивший наш ИБП и сразу подобравший для него драйвер. Берём блок [nutdev1] и добавляем его в `/etc/nut/ups.conf`
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
`[ippon]` - имя ИБП для NUT латиницей, с помощью которого и будем производить все манипуляции с ИБП </br>
`driver` - драйвер ИБП </br>
`port` - порт, на котором висит ИБП (для подключения через USB указываете значение "auto". Для snmp-ups: имя хоста SNMP агента. Для newhidups: значение "auto" для автоматического соединения с USB UPS </br>
`vendorid` и `productid` - идентификаторы устройства </br>
`product` - тип устройства. В данном случаем внутри ИБП уже установлен конвертор USB to Serial </br>
`serial` - серийный номер </br>
`vendor` - бренд изготовителя устройства </br>
`bus` - канал/шина на котором расположен порт </br>
`desc` - для удобства добавляем имя и\или расположение ИБП. Можно указывать кириллицей </br>
`override.battery.voltage.nominal = 12.00` - номинальное напряжение АКБ 12V </br>
`default.battery.voltage.high = 13.80` - напряжение для 100% заряда </br>
`default.battery.voltage.low = 10.20` - минимальное напряжение АКБ </br>

Для 12V - `override.battery.voltage.nominal = 12.00` </br>
12V x 0.15 = 1.8V </br>
12V + 1.8V = 13.8V `default.battery.voltage.high = 13.80` </br>
12V - 1.8V = 10.20V `default.battery.voltage.low = 10.20` </br>
</br>
Для 24V - `override.battery.voltage.nominal = 24.00` </br>
24V x 0.15 = 3.6V </br>
24V + 3.6V = 27.6V `default.battery.voltage.high = 27.60` </br>
24V - 3.6V = 20.4V `default.battery.voltage.low = 20.40` </br>

Чтобы сервер работал автономно, добавим в `/etc/nut/nut.conf` "MODE=standalone" </br>
```
root@proxmox:~# echo "MODE=standalone" > /etc/nut/nut.conf
```
Пробуем запускать сервер NUT: </br>
```
root@proxmox:~# systemctl start nut-server.service
```
Проверяем что данные с ИБП считываются:
```
root@proxmox:~# upsc ippon
Init SSL without certificate database
battery.charge: 100
battery.voltage: 13.60
battery.voltage.high: 13.60
battery.voltage.low: 10.40
battery.voltage.nominal: 12.00
device.type: ups
driver.name: nutdrv_qx
driver.parameter.bus: 003
driver.parameter.pollfreq: 30
driver.parameter.pollinterval: 2
driver.parameter.port: auto
driver.parameter.product: USB to Serial
driver.parameter.productid: 5161
driver.parameter.serial: 20100826
driver.parameter.synchronous: auto
driver.parameter.vendor: INNO TECH
driver.parameter.vendorid: 0665
driver.version: 2.8.0
driver.version.data: Q1 0.07
driver.version.internal: 0.32
driver.version.usb: libusb-1.0.26 (API: 0x1000109)
input.frequency: 50.1
input.voltage: 228.0
input.voltage.fault: 227.4
output.voltage: 228.0
ups.beeper.status: enabled
ups.delay.shutdown: 30
ups.delay.start: 180
ups.load: 28
ups.productid: 5161
ups.status: OL
ups.temperature: 25.0
ups.type: offline / line interactive
ups.vendorid: 0665
```
Добавляем пользователя для upsd в файле `/etc/nut/upsd.users`
```
[upsmon]
        password = upsmon
        actions = SET
        instcmds = ALL
        upsmon master
```
Атрибуты: </br>
`[upsmon]` - имя пользователя </br>
`password` - пароль пользователя </br>
`actions` - возможности: SET - изменить значения определенных переменных в ИБП. FSD - установка флага "принудительного выключения" для ИБП. </br>
`instcmds` - разрешения пользователю на инициирование конкретных команд. Применяя "ALL" вы разрешаете использовать все команды, Существует множество команд выполните "upscmd -l  <имя ИБП в настройках>" чтобы увидеть, что ваше оборудование поддерживает. Вот, к примеру, несколько команд: </br>
`test.panel.start` - старт теста передней панели </br>
`test.battery.start` - старт теста батареи </br>
`test.battery.stop` - остановка теста батареи </br>
`calibrate.start` - запуск калибровки батареи </br>
`calibrate.stop` - остановка калибровки батареи </br>
`upsmon master` - полные полномочия управления питанием подключенных к системе UPS. Отвечает за выключение разряженного аккумулятора. Выключение происходит после безопасного выключения всех slave мониторов. Если ваш ИБП подключен непосредственно к системе через последовательный порт, то для upsmon этой системы следует определить его как master. `upsmon slave` - эта система, под управлением upsmon master и она не выключается непосредственно. Операционная система будет выключена перед отключением питания master. Используйте этот режим при запуске монитора на других серверах работающих на том же ИБП. И очевидно, что только один сервер может быть подключен к последовательному порту на UPS, коим будет является master. Все остальные сервера будут slave. `upsmon monitor-only` - при этом режиме будут создаваться уведомления о состоянии или изменении работы батареи, переключении на линию и т.д., но не будет завершать работу системы.
