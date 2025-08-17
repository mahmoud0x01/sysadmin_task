# Задача системного администратора

1 - Сначала создаем окружение для хранения устройств виртуализации и загрузки ISO `Я выбрал USB-устройство`
```bash
USB_DEVICE="/dev/sdb1"  # Отрегулируйте под ваш USB раздел
sudo umount $USB_DEVICE 2>/dev/null || true
sudo mkfs.ext4 -F $USB_DEVICE -L "VM-Storage"
# Создаем точку монтирования и монтируем
sudo mkdir -p /mnt/usb-vms
sudo mount $USB_DEVICE /mnt/usb-vms
# Устанавливаем разрешения
sudo chown $USER:$USER /mnt/usb-vms
chmod 755 /mnt/usb-vms
clear
echo "✅ USB-диск подготовлен в /mnt/usb-vms"
```

2 - Во-вторых, мы загрузили ISO Windows Server 2012
```bash
cd /mnt/usb-vms/iso
# Загружаем Windows Server 2019 Evaluation
wget -O windows_server_2019.iso "https://software-download.microsoft.com/download/pr/17763.737.190906-2324.rs5_release_svc_refresh_SERVER_EVAL_x64FRE_en-us_1.iso"
```

3 - Создаем временную сеть для ВМ (без сохранения, только временно) и запускаем её  IPV6_PREFIX=YOURS if you have 
```bash
IPV6_PREFIX="YOUR_IPV6_PREFIX::/64"
# Безопасно удаляем подсеть и завершающие двоеточия
BASE6=$(echo ${IPV6_PREFIX%/*} | sed 's/::$//')
sudo virsh net-define /dev/stdin <<EOF
<network>
 <name>usb-vm-network</name>
<forward mode='nat'/>
 <bridge name='virbr-usb' stp='on' delay='0'/>
 <ip address='192.168.150.1' netmask='255.255.255.0'>
 <dhcp>
   <range start='192.168.150.10' end='192.168.150.100'/>
 </dhcp>
</ip>
ip family='ipv6' address='${BASE6}:150::1' prefix='64'>
<dhcp>
 <range start='${BASE6}:150::10' end='${BASE6}:150::100'/>
 </dhcp>
 </ip>
 </network>
EOF
# Временно запускаем сеть
sudo virsh net-start usb-vm-network
echo "✅ Временная сеть создана (будет удалена при перезагрузке)"
sudo virsh net-list --all
```

4 - Создаем виртуальное дисковое устройство Qemu
```bash
# Создаем диск ВМ на USB
qemu-img create -f qcow2 /mnt/usb-vms/vms/windows-demo.qcow2 40G
```

5 - Запускаем виртуальную систему Windows Server
```bash
virt-install --connect qemu:///system   --name demo-rdp-server   --ram 8192   --vcpus 4   --disk path=/mnt/usb-vms/vms/windows-demo.qcow2,format=qcow2,bus=virtio,cache=none   --cdrom /mnt/usb-vms/iso/windows_server_2019.iso   --network network=usb-vm-network,model=virtio   --graphics spice,listen=0.0.0.0   --video qxl   --os-variant win2k19   --boot cdrom,hd   --noautoconsole
```

6 - Подключаем ISO с драйверами virtio, чтобы система Windows могла видеть виртуальный жесткий диск **это было исправлением ошибки**
```bash
sudo virt-xml demo-rdp-server   --add-device   --disk path=/mnt/usb-vms/iso/virtio-win.iso,device=cdrom
```

7 - Доступ к виртуальному серверу через RDP
```bash
virt-viewer --connect qemu:///system demo-rdp-server
```

8 - Включение RDP для 100 пользователей на Windows ВМ
```bash
# Выбираем первый сетевой адаптер и устанавливаем статический IP (настройте по необходимости)
$adapter = Get-NetAdapter | Select-Object -First 1
New-NetIPAddress -InterfaceIndex $adapter.InterfaceIndex -IPAddress 192.168.200.50 -PrefixLength 24 -DefaultGateway 192.168.200.1
Set-DnsClientServerAddress -InterfaceIndex $adapter.InterfaceIndex -ServerAddresses 8.8.8.8

# Переименовываем компьютер для демонстрации
Rename-Computer -NewName "DEMO-RDP" -Force

# Включаем RDP и брандмауэр
Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" -name "fDenyTSConnections" -value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# Включаем одновременные сессии (режим демонстрации)
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\Licensing Core" -Name "EnableConcurrentSessions" -PropertyType DWord -Value 1 -Force

# Создаем 100 демонстрационных пользователей
1..100 | ForEach-Object {
    $username = "demo{0:D3}" -f $_   # demo001, demo002, ...
    $password = ConvertTo-SecureString "Demo123!" -AsPlainText -Force
    New-LocalUser -Name $username -Password $password -PasswordNeverExpires -ErrorAction SilentlyContinue
    Add-LocalGroupMember -Group "Remote Desktop Users" -Member $username -ErrorAction SilentlyContinue
}

Write-Host "✅ 100 демонстрационных пользователей создано (demo001-demo100)"
Write-Host "🔑 Пароль для всех: Demo123!"
```

9 - Установка Zabbix на хост с использованием **Docker**
```bash
# Создаем рабочую папку на USB
mkdir -p /mnt/usb-vms/zabbix
cd /mnt/usb-vms/zabbix

# Создаем docker-compose.yml
cat > docker-compose.yml <<EOF
version: '3.8'

services:
  zabbix-db:
    image: mysql:8.0
    container_name: demo-zabbix-db
    environment:
      MYSQL_ROOT_PASSWORD: demo
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: demo
    volumes:
      - ./zabbix-data:/var/lib/mysql
    command: >
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_bin
      --default-authentication-plugin=mysql_native_password
      --log-bin-trust-function-creators=1
    ports:
      - "3306:3306"

  zabbix-server:
    image: zabbix/zabbix-server-mysql:alpine-latest
    container_name: demo-zabbix-server
    environment:
      DB_SERVER_HOST: zabbix-db       # <--- используем имя сервиса
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: demo
    depends_on:
      - zabbix-db
    ports:
      - "10051:10051"

  zabbix-web:
    image: zabbix/zabbix-web-apache-mysql:alpine-latest
    container_name: demo-zabbix-web
    environment:
      DB_SERVER_HOST: zabbix-db       # <--- используем имя сервиса
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: demo
      ZBX_SERVER_HOST: demo-zabbix-server
    depends_on:
      - zabbix-server
    ports:
      - "8080:8080"
EOF

# Запускаем Zabbix
docker-compose up -d
```

10 - Настройка Zabbix с Windows
![Скриншот](Screenshot%20From%202025-08-18%2003-01-38.png)

11 - Доступ к веб-интерфейсу Zabbix с хоста
![Скриншот](Screenshot%20From%202025-08-18%2003-15-09.png)
