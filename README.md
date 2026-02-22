# Приручение ИБП DEXP / DNS (ID 0001:0000) в Proxmox VE (Debian)

Если вы купили бюджетный ИБП (DEXP, DNS, Powercom, Ippon), в моем случае это **DEXP MIX 850VA**, и при подключении по USB система радостно определяет его как `Fry's Electronics (0001:0000)` — вас ждут веселые выходные.

Я не нашёл толковой информации в сети. Единственное, что было известно: на Windows железка работает через проприетарный софт `UPSmartUi` по протоколу `Mega(USB)`. Отправив письмо производителю на электронную почту, я получил ожидаемый ответ: *"Поддержка в Linux-дистрибутивах не заявлена, только Windows"*. И всё. Больше никакой информации. Поэтому было решено разобраться с этим Франкенштейном самостоятельно, методом проб, ошибок и чтения исходников.

Стандартный драйвер NUT (`blazer_usb` или `nutdrv_qx`) с этой железкой нормально работать не будет. Симптомы классические:
* Постоянные отвалы (`Communications lost`).
* Ошибки `Can't claim USB device [0001:0000]@0/0: Other error`.
* Ошибка `Entity not found`.

**Почему это происходит:**
Ядро Linux видит в этом китайском контроллере HID-устройство (условную клавиатуру) и намертво перехватывает его драйвером `usbhid`. Вдобавок сам контроллер шлет ответы кривой длины, от чего дефолтный 8-байтный буфер драйвера `blazer_usb` просто давится и падает.

Здесь собран рабочий, хардкорный метод, как заставить это работать стабильно: изоляция от ядра, патч буфера драйвера из исходников и ручная инициализация.

![orig (3)](https://github.com/user-attachments/assets/a28c3a31-bdf5-45a3-989f-2af632654a38)

---

## 🛠 Пошаговый мануал

Выполнять от пользователя `root`. В командной оболочке самого узла.

### 1. Подготовка и зависимости
Ставим компиляторы и либы для работы с USB, чтобы собрать драйвер.
```bash
apt update
apt install -y build-essential pkg-config libusb-1.0-0-dev libusb-dev wget nano
```
### 2. Отключаем systemd
Службы NUT любят стартовать раньше времени, когда шина еще не готова. Вырубаем штатный автозапуск, будем дергать их вручную.
```bash
systemctl stop nut-driver nut-server nut-client
systemctl mask nut-driver nut-server nut-client
```
### 3. Бьем ядро по рукам (USB Quirks)
Запрещаем драйверу `usbhid` трогать наш ИБП при загрузке.
```bash
echo "options usbhid quirks=0x0001:0x0000:0x0004" > /etc/modprobe.d/usbhid.conf
update-initramfs -u
```
### 4. Патч и сборка драйвера (Спидхак на 102 байта)
Качаем сорцы NUT 2.8.1, меняем размер буфера с 8 байт на 102, компилируем и подкидываем в систему. Копипастите блок целиком:
```bash
cd /tmp
wget [https://networkupstools.org/source/2.8/nut-2.8.1.tar.gz](https://networkupstools.org/source/2.8/nut-2.8.1.tar.gz)
tar -zxvf nut-2.8.1.tar.gz
cd nut-2.8.1

# Конфигурим
./configure --with-usb --with-user=nut --with-group=nut

# Патчим blazer_usb.c (увеличиваем буферы)
sed -i 's/char reply\[8\]/char reply\[102\]/g' drivers/blazer_usb.c
sed -i 's/char buf\[8\]/char buf\[102\]/g' drivers/blazer_usb.c

# Собираем
make

# Заменяем дефолтный драйвер
cp drivers/blazer_usb /usr/lib/nut/blazer_usb
chmod +x /usr/lib/nut/blazer_usb
```
### 5. Конфиг NUT
Минимально достаточный конфиг для этого ИБП.
```bash
cat <<EOF > /etc/nut/ups.conf
[myups]
    driver = blazer_usb
    subdriver = krauler
    port = auto
    vendorid = 0001
    productid = 0000
    langid_fix = 0x0409
    desc = "DEXP MIX 850"
EOF
```
### 6. Скрипт инициализации (rc.local)
Самая мякотка. Скрипт ждет прогрузки USB, динамически находит, в какой порт вы воткнули ИБП, сбрасывает залипшие интерфейсы ядра и только потом поднимает демоны NUT.
```bash
nano /etc/rc.local
```
Вставляем (перед `exit 0`, если он там есть):
```bash
#!/bin/sh -e

# Ждем инициализацию USB-контроллера
sleep 7

# Ищем наш ИБП
USB_PORT=$(grep -l "0001" /sys/bus/usb/devices/*/idVendor 2>/dev/null | head -n 1 | cut -d/ -f6)

if [ -n "$USB_PORT" ]; then
    echo "UPS found on port: $USB_PORT. Unbinding..."
    for iface in /sys/bus/usb/devices/$USB_PORT:*; do
        if [ -e "$iface/driver/unbind" ]; then
            echo "${iface##*/}" > "$iface/driver/unbind" || true
        fi
    done
fi

# Готовим среду
mkdir -p /run/nut
rm -f /run/nut/*.pid /run/nut/blazer_usb-myups
chown root:nut /run/nut
chmod 775 /run/nut

# Старт патченного драйвера
/usr/lib/nut/blazer_usb -u root -a myups
sleep 3

# Фикс прав сокета
if [ -S "/run/nut/blazer_usb-myups" ]; then
    chown root:nut /run/nut/blazer_usb-myups
    chmod 660 /run/nut/blazer_usb-myups
fi

# Старт демонов
/usr/sbin/upsd -u root
/usr/sbin/upsmon -u root

# (Опционально) Имперский марш через системный динамик (apt install beep)
# (modprobe pcspkr || true; beep -f 440 -l 500 -n -f 440 -l 500 -n -f 440 -l 500 -n -f 349 -l 350 -n -f 523 -l 150 -n -f 440 -l 500 -n -f 349 -l 350 -n -f 523 -l 150 -n -f 440 -l 1000 -n -f 659 -l 500 -n -f 659 -l 500 -n -f 659 -l 500 -n -f 698 -l 350 -n -f 523 -l 150 -n -f 415 -l 500 -n -f 349 -l 350 -n -f 523 -l 150 -n -f 440 -l 1000) & # :D

exit 0
```
Даем права:
```bash
chmod +x /etc/rc.local
```
### 7. Финал
```bash
reboot
```
После перезагрузки проверяем, пошел ли опрос:
```bash
upsc myups battery.charge
```
Если отдало проценты (например, 100) — готово.

*PRO TIP: Если даже после этого ловите ошибки связи — просто переткните USB-кабель ИБП в другой порт (лучше всего в порт USB 2.0 напрямую на материнской плате, без хабов). Скрипт сам найдет его при следующей загрузке.*
