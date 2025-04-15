# Запускаем Openwrt в виртуалке с отдельным адресом и socks прокси на Apple silicon

#### Openwrt в UTM с использованием Apple Virtualization Framework и Bridge адаптера для WAN

  * [О чем этот текст](#о-чем-этот-текст)
  * [Коротко о Apple Virtualization Framework](#коротко-о-apple-virtualization-framework)
  * [Подготовка к установке](#подготовка-к-установке)
  * [Создание виртуальной машины](#создание-виртуальной-машины)
  * [Конфигурация виртуальной машины](#конфигурация-виртуальной-машины)
  * [Первый запуск виртуальной машины](#первый-запуск-виртуальной-машины)
  * [Конфигурация установленной системы](#конфигурация-установленной-системы)
  * [Установка socks прокси](#бонус-установка-socks-прокси)
  * [Пара лайфхаков](#пара-лайфхаков)

## О чем этот текст

В этом тексте мы разберем, как создать виртуалку с Openwrt и собственным виртуальным сетевым адаптером
(bridge) для выхода наружу.

Это будет обычная установка Openwrt с WAN, LAN и файрволом, достаточно близкая к настройкам по-умолчанию.

Мы будем использовать [UTM](https://mac.getutm.app/) с включенным чекбоксом Apple Virtualization.

Установка под UTM с QEMU уже описана [на сайте Openwrt.](https://openwrt.org/docs/guide-user/virtualization/utm)
Если нужна именно она, имеет смысл посмотреть туда. 
У QEMU богаче выбор сетевых устройств и видео, но он потребляет немного больше ресурсов.

В конце вы найдёте дополнительный раздел, зачем всё это было нужно мне. 
Мне нужен был отдельный прокси для одного приложения. У вас могут быть свои идеи.

## Коротко о Apple Virtualization Framework

В macOS на процессорах Apple silicon появился API виртуализации
[Apple Virtualization Framework,](https://developer.apple.com/documentation/virtualization)
который спроектирован специально для этих процессоров и обладает лучшей производительностью.

Подробнее про этот API и его историю можно прочитать на прекрасном ресурсе [Eclecticlight.co](https://eclecticlight.co/2023/12/29/why-are-apple-silicon-vms-so-different/)

Существует уже достаточно много разных приложений, использующих этот API. 
У каждого есть какая-то своя специализация или ниша.
Например, [OrbStack](https://orbstack.dev/) создаст в вашем любимом терминале комфортную иллюзию, 
что это такой Linux на быстром aarch64 и т.д.

[UTM](https://mac.getutm.app/) был выбран потому, что он лёгкий по занимаемым ресурсам и бесплатный 
(его версия в AppStore платная, но можно поставить с сайта официально без оплаты).

## Подготовка к установке

Для начала нам нужно скачать с сайта [openwrt.org](https://openwrt.org/) образ самого Openwrt.
Начиная с версий 24.10.0 он называется *armsr-armv8-generic*. 
Взять его можно [здесь](https://firmware-selector.openwrt.org/?version=24.10.0&target=armsr%2Farmv8&id=generic)
по первой ссылке *COMBINED-EFI (EXT4)* или сразу скачать 
[openwrt-24.10.0-armsr-armv8-generic-ext4-combined-efi.img.gz](https://downloads.openwrt.org/releases/24.10.0/targets/armsr/armv8/openwrt-24.10.0-armsr-armv8-generic-ext4-combined-efi.img.gz)

Ещё потребуется терминал, чтобы набрать несколько команд. Стандартный Terminal.app подойдёт.

К слову, есть отличный терминал [Alacritty.](https://alacritty.org/) Если к нему добавить [tmux,](https://github.com/tmux/tmux/wiki) получится вполне комфортная среда. Один из вариантов готовых настроек можно найти [здесь.](https://github.com/olegnet/shell-config-files)

Настроив терминал, можно сразу распаковать скачанный выше файл: `gunzip openwrt-24.10.0-armsr-armv8-generic-ext4-combined-efi.img.gz`

## Создание виртуальной машины

Выбираем *Файл -> Новый* или просто нажимаем *Command-N*. 

Появляется меню, где нужно выбрать "Виртуализировать".

<table border="1"><tr><td><img alt="Запуск" height="400" src="screenshots/01.png"/></td></tr></table>

На следующем экране выбираем "Linux".

<table border="1"><tr><td><img alt="Операционная система" height="400" src="screenshots/02.png"/></td></tr></table>

Здесь отмечаем чекбоксы "Использовать виртуализацию Apple" и "Загрузка с образа ядра".

<table border="1"><tr><td><img alt="Linux 1" height="400" src="screenshots/03.png"/></td></tr></table>

На этом экране есть единственный обязательный файл "Несжатое первоначальное ядро Linux".
Выбираем там скачанный выше файл с образом Openwrt. 
Если честно, позже мы ещё раз его добавим, а это место сейчас просто нужно пройти.

<table border="1"><tr><td><img alt="Выбор файла" width="600" src="screenshots/04.png"/></td></tr></table>

Вторая часть этого экрана должна выглядеть так.

<table border="1"><tr><td><img alt="Linux 2" height="400" src="screenshots/05.png"/></td></tr></table>

Выбираем память и количество ядер CPU, доступных виртуальной машине. 256 MB и 2 ядра достаточно.
При необходимости позже эти значения можно будет изменить. 
Например, на сайте Openwrt пишут, что лучше поставить 512 MB памяти.

<table border="1"><tr><td><img alt="Оборудавание" height="400" src="screenshots/06.png"/></td></tr></table>

На следующем экране просто нажимаем "Продолжить". Этот диск всё равно будем удалять.

<table border="1"><tr><td><img alt="Хранилище" height="400" src="screenshots/07.png"/></td></tr></table>

Здесь тоже нажимаем "Продолжить".

<table border="1"><tr><td><img alt="Каталог общего доступа" height="400" src="screenshots/08.png"/></td></tr></table>

Дальше выбираем название виртуальной машины и отмечаем чекбокс "Открыть параметры ВМ".

<table border="1"><tr><td><img alt="Краткая информация" height="400" src="screenshots/09.png"/></td></tr></table>

## Конфигурация виртуальной машины

В этот момент виртуальная машина создана, но мы только на половине пути.

На следующем экране можно поменять иконку на Openwrt (кнопка *Choose*).

<table border="1"><tr><td><img alt="Информация" width="600" src="screenshots/10.png"/></td></tr></table>

На экране "Загрузка" меняем Bootloader на *UEFI*.

<table border="1"><tr><td><img alt="Загрузка" width="600" src="screenshots/11.png"/></td></tr></table>

Дальше на экране Virtualization можно убрать чекбоксы у звука и буфера обмена.

<table border="1"><tr><td><img alt="Виртуализация" width="600" src="screenshots/12.png"/></td></tr></table>

Теперь можно поменять у *Последовательного* (порта) поле Режим.
Дисплей пока оставим, но потом, когда все настроим, его лучше просто удалить.

<table border="1"><tr><td><img alt="Последовательный (порт)" width="600" src="screenshots/13.png"/></td></tr></table>

У нас есть один сетевой адаптер, где создана *Общая сеть*. Это будет LAN.

Нужен ещё один сетевой адаптер *Сеть с мостом (расширенная)*. Это будет WAN.

<table border="1"><tr><td><img alt="Сеть" width="600" src="screenshots/14.png"/></td></tr></table>

Осталось удалить диск, созданный выше, и добавить образ Openwrt.

<table border="1"><tr><td><img alt="Импорт" width="600" src="screenshots/15.png"/></td></tr></table>

Нажимаем на *Новый*, там на *Импорт* и выбираем образ Openwrt, который мы скачали выше и распаковали.

UTM скопирует этот файл к себе в *~/Library/Containers/com.utmapp.UTM/Data/Documents*,
поэтому сохранять его не обязательно.

<table border="1"><tr><td><img alt="Диск" width="600" src="screenshots/16.png"/></td></tr></table>

Для настройки удобно убрать чекбокс Dynamic Resolution и поменять разрешение на то, которое будет комфортно для вас.

<table border="1"><tr><td><img alt="Дисплей" width="600" src="screenshots/17.png"/></td></tr></table>

На этом конфигурация виртуальной машины в меню UTM завершена.

Дальше самое интересное.

## Первый запуск виртуальной машины

Нажимаем на большую кнопку в центре и запускаем.

После загрузки нужно нажать Enter для появления приглашения системы.
Набираем "ip a" и видим, что eth1 (WAN) получил адрес по DHCP от роутера, что нам вполне подходит,
а c br-lan (LAN) будем разбираться дальше.

<img alt="Дисплей" width="600" src="screenshots/18.png"/>

Теперь можно открыть терминал в macOS и запустить там команду `ifconfig`.
Где-то ближе к концу выдачи должны быть наши новые интерфейсы. Нас интересует адрес *192.168.64.1*, который видим у появившегося *bridge100*.
В принципе, можно сразу прописать нашему интерфейсу LAN статический адрес из этой сети, но не будем торопиться.

```
vmenet0: flags=8963<UP,BROADCAST,SMART,RUNNING,PROMISC,SIMPLEX,MULTICAST> mtu 1500
        options=60<TSO4,TSO6>
        ether 42:e0:cf:10:ed:fb
        media: autoselect
        status: active
vmenet1: flags=8963<UP,BROADCAST,SMART,RUNNING,PROMISC,SIMPLEX,MULTICAST> mtu 1500
        options=60<TSO4,TSO6>
        ether 56:7a:f9:29:d0:bb
        media: autoselect
        status: active
bridge100: flags=8a63<UP,BROADCAST,SMART,RUNNING,ALLMULTI,SIMPLEX,MULTICAST> mtu 1500
        options=63<RXCSUM,TXCSUM,TSO4,TSO6>
        ether 86:2f:57:d7:15:64
        inet 192.168.64.1 netmask 0xffffff00 broadcast 192.168.64.255
        inet6 fe80::842f:57ff:fed7:1664%bridge100 prefixlen 64 scopeid 0x1a
        inet6 fdcf:15c3:8a2e:6e59:c63:84ef:d67e:45f prefixlen 64 autoconf secured
        Configuration:
                id 0:0:0:0:0:0 priority 0 hellotime 0 fwddelay 0
                maxage 0 holdcnt 0 proto stp maxaddr 100 timeout 1200
                root id 0:0:0:0:0:0 priority 0 ifcost 0 port 0
                ipfilter disabled flags 0x0
        member: vmenet0 flags=10803<LEARNING,DISCOVER,PRIVATE,CSUM>
                ifmaxaddr 0 port 24 priority 0 path cost 0
        nd6 options=201<PERFORMNUD,DAD>
        media: autoselect
        status: active
bridge101: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
        options=63<RXCSUM,TXCSUM,TSO4,TSO6>
        ether 86:2f:57:d7:15:65
        Configuration:
                id 0:0:0:0:0:0 priority 0 hellotime 0 fwddelay 0
                maxage 0 holdcnt 0 proto stp maxaddr 100 timeout 1200
                root id 0:0:0:0:0:0 priority 0 ifcost 0 port 0
                ipfilter disabled flags 0x0
        member: en0 flags=8003<LEARNING,DISCOVER,MACNAT>
                ifmaxaddr 0 port 14 priority 0 path cost 0
        member: vmenet1 flags=10003<LEARNING,DISCOVER,CSUM>
                ifmaxaddr 0 port 25 priority 0 path cost 0
        media: autoselect
        status: active
```

Дальше можно пойти двумя путями: продолжать набирать команды в окне виртуальной машины или подключиться 
к виртуальному терминалу, где работает copy-paste.

Для этого нужно найти путь к виртуальному терминалу и запустить команду screen с этим параметром.

<img alt="Дисплей" width="600" src="screenshots/19.png"/>

В нашем случае это `screen /dev/ttys000`

Там тоже нужно нажать Enter, как нам и предлагают.

Теперь выставим запрос dhcp и для LAN.

```shell
uci set network.lan.proto='dhcp'
uci commit network
uci show network
reboot
```

После перезагрузки мы видим, что адрес у LAN 192.168.64.4.
Отлично, пропишем его статикой.

```
root@OpenWrt:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br-lan state UP qlen 1000
    link/ether 1e:7e:9a:8a:1f:a7 brd ff:ff:ff:ff:ff:ff
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP qlen 1000
    link/ether 8a:89:f2:21:9d:2f brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.133/24 brd 192.168.1.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::8889:f2ff:fe21:9d2f/64 scope link
       valid_lft forever preferred_lft forever
4: br-lan: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether 1e:7e:9a:8a:1f:a7 brd ff:ff:ff:ff:ff:ff
    inet 192.168.64.4/24 brd 192.168.64.255 scope global br-lan
       valid_lft forever preferred_lft forever
    inet6 fda6:7f8:aee7::1/60 scope global noprefixroute
       valid_lft forever preferred_lft forever
    inet6 fe80::1c7e:9aff:fe8a:1fa7/64 scope link
       valid_lft forever preferred_lft forever
```

В терминале Openwrt

```shell
uci set network.lan.proto='static'
uci set network.lan.ipaddr='192.168.64.4'
uci set network.lan.netmask='255.255.255.0'
uci commit network
uci show network
reboot
```

Здесь же нас предупреждают, что не установлен пароль для root.
Запускаем `passwd` и ставим пароль.

## Конфигурация установленной системы

Теперь у нас есть полностью сконфигурированный Openwrt.
Заходим в Luci (это web UI), вставляем в браузер адрес 192.168.64.4 и набираем пароль для root, 
который только что установили.

<table border="1"><tr><td><img alt="Дисплей" width="600" src="screenshots/20.png"/></td></tr></table>

В этом месте будет полезно настроить таймзону и выключить dhcp сервер на LAN.
Network -> Interfaces -> (для lan) Edit -> DHCP server -> General setup -> Ignore interface.
Не забудьте нажать кнопку "Save & Apply".

Там же, в Network -> Interfaces, нужно будет иногда нажимать Restart на WAN и WAN6, 
если вы перешли в другую сеть с вашим компьютером.

Дальше всё, как обычно, в свежеустановленном Openwrt.

В частности, неплохо бы обновить пакеты. В System -> Software нужно выбрать Update lists.
После этого в табе Updates будет список пакетов, которые нужно обновить.

Можно сделать это и проще. В терминале сказать

```shell
opkg update
opkg list-upgradable | awk '{print $1}' | xargs opkg upgrade
```

Сразу добавить несколько полезных утилит.

```shell
opkg install htop btop lsof 
opkg install joe nano  # <-- любой редактор по вашему вкусу
```

Кстати, терминал, где мы запускали screen, уже можно закрыть.
Вместо этого проще использовать `ssh root@192.168.64.4`

## Бонус: установка socks прокси

Мне все это нужно было, чтобы поставить socks прокси и иметь отдельный маршрут, мимо тоннелей, которые обычно подняты на моём лаптопе.

Можно было бы просто набирать в терминале что-то вроде `ssh root@192.168.64.4 -D 127.0.0.1:1080`,
но вместо этого достаточно поставить один пакет с socks прокси.

```shell
opkg install microsocks
joe /etc/config/microsocks  # вместо joe подставьте ваш любимый редактор
```

```
config microsocks 'config'
        option enabled '1'              # <-- меняем 0 на 1
        option bindaddr ''
        option listenip '192.168.64.4'  # меняем адрес на LAN
        option port '1080'
        option user ''
        option password ''
        option auth_once '0'
        option quiet '0'                # тоже меняем 1 на 0
```

Последний параметр "quiet" можно и не трогать, но для понимания, что всё работает, полезно включить 
на минуту логирование. Потом конечно лучше выключить.

```shell
/etc/init.d/microsocks restart
/etc/init.d/microsocks status
# running
netstat -an|grep 1080
# tcp        0      0 192.168.64.4:1080       0.0.0.0:*               LISTEN
```

Проверим в терминале macOS `telnet 192.168.64.4 1080`.

Запускаем отдельный браузер через этот прокси и убеждаемся, что он идёт альтернативным маршрутом.
Например, здесь [whatismyipaddress.com](https://whatismyipaddress.com/)

```shell
open -a "Google Chrome Beta" --args --proxy-server=socks5://192.168.64.4:1080
```

Теперь можно посмотреть лог командой `logread` и выключить логгирование, вернув `option quiet '1'` 
в */etc/config/microsocks*.

## Пара лайфхаков

### UTM с Openwrt можно запускать на логине пользователя. Достаточно положить этот текст в `~/Library/LaunchAgents/com.utm.openwrt.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.utm.openwrt</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Applications/UTM.app/Contents/MacOS/utmctl</string>
        <string>start</string>
        <string>openwrt</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <false/>
</dict>
</plist>
```

Окно UTM, появляющееся на старте, можно смело закрывать. 
Автоматизация закрывания этого окна остается упражнением для читателя.

### Вместо того чтобы каждый раз открывать или переключаться в терминал, можно сделать небольшой скрипт.

Открываем Script Editor (системное приложение в macOS) и пишем 
```text
do shell script "open -a 'Google Chrome Beta' --args --proxy-server=socks5://192.168.64.4:1080"
```
Далее в File -> Export нужно выбрать в File Format: Application (вместо Script) и сохранить,
как `~/Applications/Chrome Beta with Proxy.app`

Теперь это "приложение" можно добавить в Dock или запускать из Spotlight Search.

### Почему в примере везде Google Chrome Beta вместо Google Chrome

Мне удобно иметь один профиль, в котором закладки и история прозрачно синхронизируются в двух браузерах, 
и при этом они ходят в сеть разными маршрутами. 

Вместо него можно использовать любой другой. В Firefox можно даже настроить прокси один раз и просто пользоваться им
без необходимости запускать с внешними параметрами.

#### Copyright © 2025 Oleg Okhotnikov. All rights reserved.
