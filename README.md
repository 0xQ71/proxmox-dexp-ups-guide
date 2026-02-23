#!/bin/bash

sleep 15

# 1. Жестко чистим хвосты, которые мог запустить systemd
killall -9 blazer_usb upsd upsmon 2>/dev/null || true

# 2. Ищем ИБП (0001:0000)
USB_PORT=$(grep -l "0001" /sys/bus/usb/devices/*/idVendor 2>/dev/null | head -n 1 | cut -d/ -f6)

if [ -n "$USB_PORT" ]; then
    echo "ИБП найден на порту $USB_PORT. Сбрасываем драйвер ядра..."
    for iface in /sys/bus/usb/devices/$USB_PORT:*; do
        if [ -e "$iface/driver/unbind" ]; then
            echo "${iface##*/}" > "$iface/driver/unbind" || true
        fi
    done
fi

# 3. Готовим среду
mkdir -p /run/nut
rm -f /run/nut/*.pid
chown root:nut /run/nut
chmod 775 /run/nut

# 4. Запускаем связку по порядку
/usr/lib/nut/blazer_usb -u root -a myups
sleep 5

# Фикс прав сокета (критично для upsd)
if [ -S "/run/nut/blazer_usb-myups" ]; then
    chown root:nut /run/nut/blazer_usb-myups
    chmod 660 /run/nut/blazer_usb-myups
fi

/usr/sbin/upsd -u root
sleep 2
/usr/sbin/upsmon -u root

exit 0
