# ///// Ubuntu - Srv /////

- Actualizamos:
```
sudo apt update
```
```
sudo apt upgrade
```

# ---------- VBox Guest Additions ---------- #

- Paquetes requeridos
```
sudo apt install build-essential linux-headers-$(uname -r) -y
```

- Monta las Guest Additions desde la parte de Dispositivos dentro de la máquina

- Instalar las VBox Guest Additions
```
cd /media/linuxtechi/VBox_GAs_7.0.4/
```
```
sudo ./VBoxLinuxAdditions.run
```

- Reiniciamos la máquina

# ---------- Tarjeta de red ---------- #

- Comando del fichero de configuración:
```
sudo vim /etc/netplan/01-network-manager-all.yaml
```

- Fichero final:
```
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s8:
      dhcp4: false
      addresses: [192.168.0.3/24]
      routes:
        - to: default
          via: 192.168.0.254
          metric: 100
          on-link: true
      nameservers:
        addresses: [8.8.8.8] 
```

- Aplicamos nuevos cambios:
```
sudo netplan apply
```

- Probar que funciona (recibes respuesta si salió todo bien):
```
ping www.google.es
```

# ---------- DHCP ---------- #

- Verificamos las ip's:
```
ifconfig
```
- (Si no tenemos el comando "ifconfig", instalar net-tools):
```
sudo apt install net-tools
```

- Instalamos isc-dhcp-server:
```
sudo apt-get install isc-dhcp-server
```

- Cambiamos configuración para que pille la tarjeta que configuramos antes:
```
sudo vim /etc/default/isc-dhcp-server
```
- Fichero de configuración:
```
INTERFACESv4="enp0s8"
INTERFACESv6=""
```

- Entramos al archivo del DHCP:
```
sudo vim /etc/dhcp/dhcpd.conf
```
- Configuramos el DHCP:
```
subnet 192.168.0.0 netmask 255.255.255.0 {
    range 192.168.0.20 192.168.0.50;
    option domain-name-servers 192.168.0.3;
    option routers 192.168.0.1;
    default-lease-time 3600;
    max-lease-time 7200;
    host ns1 {
        hardware ethernet 08:00:27:4a:9c:92;
        fixed-address 192.168.0.5
    }
}
```

- Comprobamos la sintaxis:
```
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

- Reiniciamos el server:
```
sudo service isc-dhcp-server restart
```

- Comprobamos el estado
```
sudo service isc-dhcp-server status
```

# ---------- DNS ---------- #

- Instalar Bind9:
```
sudo apt install bind9 bind9-utils
```

- Comprobar que funciona:
```
sudo systemctl status bind9
```

- Permitir acceso:
```
sudo ufw allow bind9
```

- Configuración mínima de bind9:
```
sudo vim /etc/bind/named.conf.options
```
```
listen-on { any; };
allow-qwery { localhost; 192.168.0.0/24; };
forwarders {
        192.168.0.3;
}
dnssec-validation no;
```

- Obligar el uso de IPv4:
```
sudo vim /etc/default/named
```
```
OPTIONS="-u bind -4"
```

- Comprobar y Reiniciar:
```
sudo named-checkconf
sudo systemctl restart bind9
systemctl status bind9
```

- Agregar zonas:
```
sudo vim /etc/bind/named.conf.local
```
```
zone "ola.cu" IN {
        type master;
        file "/etc/bind/zonas/db.ola.cu";
};

zone "0.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/zonas/db.192.168.0";
};
```

- Crear carpeta donde se guardará esa configuración:
```
sudo mkdir /etc/bind/zonas
```

- Crear archivos con la configuración (primer archivo):
```
sudo vim /etc/bind/zonas/db.ola.cu
```
```
$TTL  1D
@     IN    SOA   ns1.ola.cu.   admin.ola.cu. (
      1           ; Serial
      12h         ; Refresh
      15m         ; Retry
      3w          ; Expire
      2h )        ; Negative Cache TTL

;     Registros NS

      IN    NS    ns1.ola.cu.
ns1   IN    A     192.168.0.3
www   IN    A     192.168.0.3
cliente1 IN A     192.168.0.5
```

- Crear archivos con la configuración (segundo archivo):
```
sudo vim /etc/bind/zonas/db.192.168.0
```
```
$TTL  1d ;
@     IN    SOA   ns1.ola.cu.   admin.ola.cu. (
                  20210222      ; Serial
                  12h           ; Refresh
                  15m           ; Retry
                  3w            ; Expire
                  2h    )        ; Negative Cache TTL

;
@     IN    NS    ns1.ola.cu.
1     IN    PTR   www.ola.cu.
```

- Comprobar archivos de zoans:
```
sudo named-checkzone ola.cu /etc/bind/zonas/db.ola.cu
sudo named-checkzone db.0.168.192.in-addr.arpa /etc/bind/zonas/db.192.168.0
```

- Reinicio:
```
sudo systemctl restart bind9
```

- Comprobar funcionamiento:
```
ping www.ola.cu
```

# ---------- WEB ---------- #

- Instalar apache2:
```
sudo apt install apache2
```

- Cambiar firewall:
```
sudo ufw allow 'Apache'
```

- Comprobar stado:
```
sudo ufw status
```

- Estado de apache:
```
sudo systemctl status apache2
```

## Verificar que está activo, meterse en la web escribiendo: http://192.168.0.3

- Crear el directorio de tu web:
```
sudo mkdir /var/www/ola.cu
```

- Dar privilegios:
```
sudo chown -R $USER:$USER /var/www/ola.cu
```

- Crear el archivo de tu web:
```
sudo vim /var/www/ola.cu/index.html
```
```
<html>
  <head>
    <title>Welcome to ola.cu</title>
  </head>
  <body>
    <h1>It works!</h1>
  </body>
</html>
```

- Crear configuracion para host:
```
sudo vim /etc/apache2/sites-available/ola.cu.conf
```
```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName ola.cu
    ServerAlias www.ola.cu
    DocumentRoot /var/www/ola.cu
    ErrroLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

- Activar y desactivar archivos:
```
sudo a2ensite ola.cu.conf
sudo a2dissite 000-default.conf
```

- Crear un script:
```
sudo vim script.sh
```
```
#!/bin/sh

echo -n Aplicando Reglas de Firewall...

iptables -F
iptables -X
iptables -Z
iptables -t nat -F

iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -t nat -P PREROUTING ACCEPT
iptables -t nat -P POSTROUTING ACCEPT

/sbin/iptables -A INPUT -i lo -j ACCEPT

iptables -A INPUT -s 192.168.1.0/24 -i enp0s8 -j ACCEPT

iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o enp0s3 -j MASQUERADE

echo 1 > /proc/sys/net/ipv4/ip_forward
```

- Ejecutar el script:
```
sudo bash script.sh
```

- Comprobar errores:
```
sudo apache2ctl configtest
```

- Reiniciar apache:
```
sudo systemctl restart apache2
```

## Verificar que se ha cambiado buscando "www.ola.cu" en un navegador

# Nota #

- Cambiar el resolv
```
sudo vim /etc/resolv.conf
```
```
nameserver 192.168.0.3
```

# ///// FIN ///// #
