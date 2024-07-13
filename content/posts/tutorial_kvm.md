+++
title = 'Организация виртуальной лаборатории с использованием KVM'
date = 2024-07-11T21:14:26+10:00
draft = false
+++
В этой статье показана настройка лаборатории для исследования вредоносного ПО с использованием KVM. Так же, как и VirtualBox или VMWare, KVM через Spice предлагает гостевые дополнения, позволяющие переносить образцы ПО с хост-компьютера на виртуальные машины. В отличие от VirtualBox или VMWare, KVM позволяет модифицировать код QEMU, что в свою очередь позволяет создавать более скрытные машины для запуска малвари.

## Сетевая инфраструктура

Возможность обеспечения виртуальной машины для анализа с доступом в интернет является важной составляющей полноценного анализа, поскольку многие современные вредоносные программы используют интернет-соединение для различных целей. Однако предоставление доступа к интернету требует дополнительных мер безопасности для защиты сети.

> [!TIP]

> Следует поместить хост-машину в DMZ. Большинство домашних маршрутизаторов обладают настройками DMZ.

Общая схема сети представлена ниже:

```
  ┌─────────┐
  │Интернет │
  └────┬────┘
       │
┌──────┴───────┐
│Хост-машина   │ Виртуальная машина (KVM) в DMZ
└──────┬───────┘
       │
    ┌──┴──┐
    │NAT  │ (по умолчанию)
    └──┬──┘
       │
   ┌───┴────┐ ┌─────────┐
   │PFSense ├─┤Изолиров.│ LAN 10.0.1.1/24 (vmbr0)
   └────────┘ └────┬────┘
	         │
	      ┌──┴───┐
	      │Remnux│ Статический анализ и перехват трафика
	      └──┬───┘
	         │
	     ┌───┴─────┐
	     │Изолир.  │ Аналитическая LAN 10.0.2.1/24 (vmbr1)
	     └──┬──────┤
	        │      │
         ┌────────┴───┐ ┌┴───────┐
         │Аналит. ВМ  │ │Windows │ Динамический анализ
         └────────────┘ └────────┘
```

**Динамический анализ** будет проводиться на машине  **Windows**. Использование **Linux** (Remnux) позволит нам применять статические аналитические инструменты и осуществлять перехват зашифрованного трафика благодаря расшифровке **TLS-секретов**. Также мы можем создавать временные **C2-соединения** для воспроизведения сценариев атаки.
В связи с особыми соображениями безопасности, сеть настроена следующем образом:
- **Remnux** и **Windows** взаимодействуют через внутреннюю сеть (internal network).
- **Remnux** получает выход в интернет через **pfSense**, выступающий как маршрутизатор и брандмауэр.
- С помщью **NAT** и **IP Forwarding** с **Remnux**-машины передается трафик **Windows**-машине через **tun-интерфейс**. Предпологается использование **VPN - соединения** во избежание утечки реальных ip-адресов.

> [!NOTE]
Без включённого VPN на машине Remnux, в Windows не будет доступа к интернету.

Далее представлен процесс установки необходимых компонентов.
## Установка KVM

* В первую очередь необходимо проверить, включена ли виртуализация на вашем хосте.

```bash
grep -Po '(vmx|svm)[^\x20]+' /proc/cpuinfo | sort | uniq # Для хост-машины
cat /sys/module/kvm_intel/parameters/nested              # Влож. виртуализ. (intel)
cat /sys/module/kvm_amd/parameters/nested                # Влож. виртуализ. (amd)
```

* Включаем вложенную виртуализацию (необязательно)
```bash
sudo nano /etc/modprobe.d/kvm.conf # Файл конфигурации вложенной виртуализации
# Добавьте следующую конфигурацию
: '
options kvm_intel nested=1
' # сохраните и перезагрузите
sudo reboot
```
Если после выполнения предыдущей команды есть какой-либо вывод, значит виртуализация включена, в противном случае включите её в BIOS и попробуйте снова. 

> [!WARNING]
Использование вложенной виртуализации приведёт к значительным потерям производительности. Если у вас слабый ПК, используйте Linux с KVM на голом железе.

* Устанавливаем зависимости
```bash
sudo apt update
sudo apt install -y virt-manager qemu-kvm build-essential python-is-python3
```

* Проверим, запущен ли **libvirtd**
```bash
sudo systemctl enable libvirtd
sudo systemctl start libvirtd
```
- Добавим текущего пользователя в **libvirtd** - группу
```bash
sudo usermod -a -G libvirt $USER
```
- Установим модифицированную версию [SeaBios](https://www.seabios.org/SeaBIOS) (изменен **/src/config.h** на более реалистичные значения)
```bash
git clone https://github.com/c3rb3ru5d3d53c/seabios.git
cd seabios/
make
sudo cp /usr/share/seabios/bios.bin /usr/share/seabios/bios.bin.bak
sudo cp out/bios.bin /usr/share/seabios/bios.bin
```
- Если вы хотите чтобы **virt-manager** имел доступ к вашей домашней директории, установите следующее разрешение (этот пункт необязателен)

```bash
sudo setfacl -m u:libvirt-qemu:rx /home/$USER
```
- Выполните перезагрузку

```bash
sudo reboot
```

## Создание виртуальных машин

В этой секции показано как создавать виртуальные машины KVM с использованием **virt-manager**.

### Создание виртуальной сети

**Настройка PFSense LAN**

1. Откройте **Edit**.
2. Перейдите к **Connection Details**.
3. Нажмите на **+** для добавления новой конфигурации.
4. Выберите **vmbr0 Isolated (PFSense LAN)**.
5. Отключите **IPV4** и **IPv6** (настройки  PFSense).

**Настройка Analysis LAN**

1. Откройте **Edit**.
2. Перейдите к **Connection Details**.
3. Нажмите на **+** для добавления новой конфигурации.
4. Выберите **vmbr1 Isolated (Analysis LAN)**.
5. Отключите **IPV4** и **IPv6** (настройки  Remnux).

### Создание виртуальной машины с PFSense

Основная цель создания машины с PFSense - это простой способ управления брандмауэром между сетью и интернетом.
* Скачать PFSense:  https://www.pfsense.org/download/
* Создадим образ **qcow2**
```bash
qemu-img create -f qcow2 pfsense.qcow2 16G
```
* Создадим виртуальную машину
	1. Импортируем существующий образ диска
	2. Выберем **pfsense.qcow2** в качестве хранилища
	3. В качестве ОС выберем **FreeBSD** 
	4. Название: **pfsense**

* Настроим конфигурацию
	1. Сетевая карта - NAT
	2.  Add Hardware -> Network -> vmbr0 (Сеть PFSense)
	3.  Add Hardware -> Network -> CDROM Device -> Manager -> PFSense ISO
	4.  Boot Options -> Установим CDROM device как первый на этапе загрукзи ОС
* Запустим машину
* В настройках PFSense установим LAN **10.0.1.1/24**

### Создание виртуальной машины Remnux (Ubuntu 24.04)

Основная цель этой виртуальной машины -  проводить статический анализ бинарных файлов с  помощью таких инструментов как IDA, Ghidra, Cutter и.т.д, а также служить шлюзом для подсети анализа. Это дает полный контроль над серверами DNS и DHCP для перехвата трафика и других операций.

>[!NOTE]
Официальные дистрибутивы Remnux обновляются довольно редко. Рекомендуется скачать последний LTS-образ Ubuntu и использовать Remnux в качестве контейнера.
* Скачать https://releases.ubuntu.com/noble/
* Создадим образ диска
```bash
qemu-img create -f qcow2 remnux.qcow2 40G
```
* Создадим новую виртуальную машину
	1. Импортируем существующий образ диска
	2. В качестве хранилище выберем **remnux.qcow2**
	3. В качестве ОС выберем Ubuntu
* Настроим конфигурацию перед установкой
	* Установим первый сетевой интерфейс как **vmbr0** (Сеть PFSense)
	* Add Hardware -> Network -> **vmbr1** (сеть анализа)
	* Add Hardware -> Storage -> CDROM Device-> Manage -> Ubuntu ISO Image
	* Boot Options -> Установим CDROM device как первый при загрузке
* Запустим ВМ
* Установим зависимости
```bash
sudo apt update
sudo apt install dnsmasq isc-dhcp-server openvpn docker.io spice-vdagent
sudo systemctl enable spice-vdagent
sudo systemctl enable docker
sudo usermod -a -G docker $USER
# выйти и зайти снова
```
* Создадим **remnux** - контейнер
```bash
docker run --rm -it -v $(pwd):$HOME -u remnux remnux/remnux-distro:focal bash
```
* Настроим интерфейсы
```bash
sudo nano /etc/netplan/01-network-manager-all.yaml
# Заменить содержимое следующим
 'network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: yes
    enp7s0:
      addresses:
        - 10.0.2.1/24
'
```

* Отключим конфликты **dnsmasq** с **systemd-resolved**
```bash
sudo nano /etc/systemd/resolved.conf
# Добавить DNSStubListener=no в конец файла и сохранить
sudo systemctl restart systemd-resolved
sudo systemctl enable dnsmasq
sudo systemctl restart dnsmasq
```
* Настроим dhcp - сервер для сети анализа
```bash
sudo nano /etc/dhcp/dhcpd.conf
# Добавить следющее в файл конфигурации
 'subnet 10.0.2.0 netmask 255.255.255.0 {
   option domain-name "mydomain.org";
   default-lease-time 600;
   max-lease-time 7200;
   option routers 10.0.2.1;
   option subnet-mask 255.255.255.0;
   option broadcast-address 10.0.2.255;
   option domain-name-servers 10.0.2.1;
   range 10.0.2.2 10.0.2.254;
}'
sudo systemctl enable isc-dhcp-server
sudo systemctl restart isc-dhcp-serve
```
* Настроим firewall PFSense. 
	1. Перейдем по адресу https://10.0.1.1  и авторизуемся **(admin:pfsense)**
	2. Добавим правила, запрещающие доступ из **LAN** в **192.168.0.1/24**
	3. Применим изменения
* Настроим internet forwarding
```bash
sudo iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE # 
sudo apt install iptables-persistent # "Yes" в обоих случаях
sudo nano /etc/sysctl.conf # Убрать комментарий с net.ipv4.ip_forward=1
sudo sysctl -w net.ipv4.ip_forward=1
```
* Создадим скрипт, позволяющий запускать VPN
```bash
nano vpn.sh
# Добавить след. инструкции и сохранить
'
#!/usr/bin/env bash
openvpn --daemon --config VPNJantit_All-servers_UDP-443.ovpn
'
chmod +x vpn.sh
sudo ./vpn.sh
```
* Установим утилиту для просмотра https трафика **MITMProxy**
```bash
sudo apt update
sudo apt install -y wireshark tshark # Yes
sudo usermod -a -G wireshark $USER   # Выйти и зайти снова
wget https://snapshots.mitmproxy.org/8.1.0/mitmproxy-8.1.0-linux.tar.gz
tar -xzvf mitmproxy-8.1.0-linux.tar.gz
sudo mv mitmdump /usr/bin/
sudo mv mitmproxy /usr/bin/
sudo mv mitmweb /usr/bin/
wget https://gist.githubusercontent.com/c3rb3ru5d3d53c/d9eb9d752882fcc630d338a6b2461777/raw/f56cbef4b98c7bad8f265534eabf696923b649a2/mitmpcap
chmod +x mitmpcap
sudo mv mitmpcap /usr/bin/
wget https://gist.githubusercontent.com/c3rb3ru5d3d53c/3bc8041a182467ccae0207394c1e16b3/raw/33bf201da721ae8f8dc057480221a3c6612e7c11/mitmhttp
chmod +x mitmhttp
sudo mv mitmhttp /usr/bin/
sudo mkdir /root/.mitmproxy
```
После этого можно перезапустить машину и использовать интернет. 

### Создание виртуальной машины Windows

Эта ВМ будет находиться в сети анализа, основная её задача  - динамический анализ исполняемых файлов (Отладка, просмотр API - вызовов и т.д.).

> [!WARNING]
> При установке Windows необходимо убедиться что загрузочным диском является **VirtIO**

- Скачаем [VirtIO ISO](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso)
- Создадим образ диска
```bash
qemu-img create -f qcow2 windows.qcow2 128G
```
* Создадим новую виртуальную машину
	1. Импортируем существующий образ диска
	2. Выберем  **windows.qcow2** в качестве хранилища
	3. Выберем Windows в качестве ОС
* Настроим конфигурацию перед установкой
	1. Установим первый сетевой интерфейс как **vmbr1** (Сеть анализа)
	2. Add Hardware -> Storage -> CDROM Device-> Manage -> Windows ISO Image
	3. Add Hardware -> Storage CDROM Device -> Manage -> VirtIO ISO
	 4. Boot Options -> Установим CDROM device как первый при загрузке
	 5. Извлечем SATA диск и заменим на VirtIO (windows.qcow2)
* Запустим установку

* Во время установки Windows диск VirtIO не будет отображаться, пока вы не перейдете к ISO-образу VirtIO и не выберете папку с драйверами для вашей версии Windows. После завершения вы можете продолжить установку.
* После установки Windows откройте "Диспетчер устройств" и найдите все драйверы со знаками вопроса, затем обновите их, используя CD-ROM с ISO-образом VirtIO.
* Скачайте и установите [Spice Guest Tools](https://www.spice-space.org/download/windows/spice-guest-tools/spice-guest-tools-latest.exe)
* Перезагрузите ВМ
После того как вы завершите все необходимые шаги, рекомендуется удалить CD-ROM привод для драйверов VirtIO.

### Дешифрование TLS - сертификатов

Этот раздел о том, как настроить дешифрование TLS на виртуальной машине Remnux для дешифровки TLS и просмотра секретов и pcap-файлов с аналитической сети (при условии, что устройства имеют установленный корневой сертификат CA).

На Remnux - машине выполним
```bash
sudo mitmhttp -i <analysis-network-interface> --enable
mitmproxy --mode transparent --listen-host 0.0.0.0 --listen-port 8080
```

На машине Windows откроем браузер и перейдем на сайт **mitm.it**, затем следуем инструкциям установки.
Ручная установка:

1. На веб-странице скачайте сертификат для Windows.
2. Выберете файл P12, чтобы начать мастер импорта.
3. Выберите расположение хранилища сертификатов. Это определяет, кто будет доверять сертификату — только текущий пользователь  или все пользователи. Нажмите "Далее".
4. Нажмите "Далее" снова.
5. Оставьте поле пароля пустым и нажмите "Далее".
6. Выберите "Разместить все сертификаты в следующем хранилище", затем нажмите "Обзор" и выберите "Доверенные корневые центры сертификации". Нажмите "ОК" и "Далее".
7. Нажмите "Завершить".
8. Нажмите "Да", чтобы подтвердить диалоговое окно предупреждения.

Автоматическая установка:

```cmd
certutil.exe -addstore root mitmproxy-ca-cert.cer
```

В вашем браузере перейдите на [https://example.com](https://example.com/), чтобы проверить успешное дешифрование в интерфейсе mitmproxy.
Если вышеуказанные шаги были успешными, теперь вы можете использовать инструмент mitmpcap для просмотра трафика в реальном времени с помощью mitmproxy, а также использовать его для захвата TLS-секретов и pcap для последующего анализа.
```bash
mitmpcap -i enp7s0 -w dump.pcap -m transparent -p 8080 -s secrets.txt -v 1 -f libpcap
```
Как только вы закончите перехват, нажмите Q, а затем Y для выхода из mitmproxy. После этого вы должны заметить, что у вас есть файлы dump.pcap и secrets.txt. Следующие шаги о том, как использовать Wireshark для анализа перехваченного трафика.
1. Откройте dump.pcap в Wireshark.  
2. Перейдите в меню Редактировать -> Настройки -> Протоколы -> TLS -> Имя файла журнала (предварительного) основного секрета, нажмите Обзор -> выберите secrets.txt.  
3. В строке фильтра введите http и нажмите Enter.  
Теперь у вас должен быть расшифрованный TLS-трафик.
Как только вы завершите захват трафика и больше хотите остановить перехват, отключите перенаправление, выполнив следующее
```bash
sudo mitmhttp -i <analysis-network-interface> --disable
```
