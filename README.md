# 2do_Parcial_Servicios-Telematicos

# Primer Punto:

# Configuración Actualizada con las Nuevas IPs

Entiendo que las IPs han cambiado:
- **Cliente (FileZilla)**: 192.168.50.2
- **Servidor 1 (Firewall UFW)**: 192.168.50.3
- **Servidor 2 (vsftp seguro)**: 192.168.50.4


## Configuración del Servidor 2 (vsftp seguro)

### Instalación y configuración

```bash
# Actualizar sistema
sudo apt update && sudo apt upgrade -y

# Instalar vsftpd
sudo apt install vsftpd -y

# Crear usuario FTP
sudo adduser ftpuser

# Configurar vsftpd
sudo nano /etc/vsftpd.conf
```


### Configuración del archivo `/etc/vsftpd.conf`

```ini
# Habilitar SSL/TLS
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES

# Certificados SSL
rsa_cert_file=/etc/ssl/certs/vsftpd.crt
rsa_private_key_file=/etc/ssl/private/vsftpd.key

# Configuración básica
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES

# Usuario específico (si creaste uno)
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO

# Modo pasivo para firewall
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=50000
pasv_address=192.168.50.3 

# Restricciones de seguridad
chroot_local_user=YES
allow_writeable_chroot=YES
```

### Agregar usuario a la lista permitida
```bash
echo "ftpuser" | sudo tee -a /etc/vsftpd.userlist
```

### Generar certificados SSL
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/private/vsftpd.key \
-out /etc/ssl/certs/vsftpd.crt

# Ajustar permisos
sudo chmod 600 /etc/ssl/private/vsftpd.key
sudo chmod 600 /etc/ssl/certs/vsftpd.crt
```

### Reiniciar servicio
```bash
sudo systemctl restart vsftpd
sudo systemctl enable vsftpd
sudo systemctl status vsftpd
```

### Configuración del Servidor 1 (Firewall UFW) 

### Configuración de UFW y redirección

```bash
# Instalar UFW
sudo apt update && sudo apt install ufw -y

# Permitir tráfico FTP en puerto 21
sudo ufw allow 21/tcp

# Habilitar IP forwarding
echo 'net.ipv4.ip_4=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Configurar NAT y redirección de puertos
sudo iptables -t nat -A PREROUTING -p tcp --dport 21 -j DNAT --to-destination 192.168.50.4:21
sudo iptables -A FORWARD -p tcp -d 192.168.50.4 --dport 21 -j ACCEPT

# Para el modo pasivo de FTP, redirigir el rango de puertos
sudo ufw allow 40000:50000/tcp
sudo iptables -t nat -A PREROUTING -p tcp --dport 40000:50000 -j DNAT --to-destination 192.168.50.4
sudo iptables -A FORWARD -p tcp -d 192.168.50.4 --dport 40000:50000 -j ACCEPT

# SNAT para el tráfico de retorno
sudo iptables -t nat -A POSTROUTING -j MASQUERADE

# Hacer permanentes las reglas
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```


### Habilitar UFW
```bash
sudo ufw enable
sudo ufw status verbose
```


# **Conexión FTP desde Terminal con SSL/TLS**

Te muestro cómo conectarte usando clientes FTP por terminal que soportan SSL/TLS:

##  Usar `lftp` **

**Instalar lftp:**
```bash
sudo apt update && sudo apt install lftp -y
```

**Conexión FTPS explícito:**
```bash
lftp -e "set ftp:ssl-force true; set ssl:verify-certificate no;" -u ftpuser,ftpuser ftp://192.168.50.3:21
```


# Segundo Punto


Paso 1. Instalar BIND9

En maestro (192.168.50.3) y esclavo (192.168.50.4):

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc -y
```

Paso 2. Configurar Maestro (192.168.50.3)

Edita `/etc/bind/named.conf.local`:

```bash
zone "servicios.com" {
    type master;
    file "/etc/bind/db.servicios.com";
    allow-transfer { 192.168.50.4; };   # Solo el esclavo puede replicar
};
```

Crea la zona:

```bash
sudo cp /etc/bind/db.local /etc/bind/db.servicios.com
sudo nano /etc/bind/db.servicios.com
```

Contenido del archivo `/etc/bind/db.servicios.com`:

```bash
$TTL    604800
@       IN      SOA     ns1.servicios.com. admin.servicios.com. (
                        3         ; Serial
                        604800    ; Refresh
                        86400     ; Retry
                        2419200   ; Expire
                        604800 )  ; Negative Cache TTL
;
@       IN      NS      ns1.servicios.com.
@       IN      NS      ns2.servicios.com.

ns1     IN      A       192.168.50.3
ns2     IN      A       192.168.50.4
www     IN      A       192.168.50.5
```

Paso 3. Configurar Esclavo (192.168.50.4)

Edita `/etc/bind/named.conf.local`:

```bash
zone "servicios.com" {
    type slave;
    masters { 192.168.50.3; };
    file "/var/lib/bind/db.servicios.com";
};
```

Paso 4. Configurar Firewall (192.168.50.5)

Permitir solo al cliente y servidores:

```bash
sudo ufw allow from 192.168.50.2 to any port 53
sudo ufw allow from 192.168.50.3 to any port 53
sudo ufw reload
```

Si quieres redirigir todas las consultas DNS del cliente al maestro/esclavo:
Editar `/etc/ufw/before.rules` en servidor (192.168.50.5):

```bash
*nat
:PREROUTING ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]

# Redirigir DNS (puerto 53) hacia maestro
-A PREROUTING -p udp --dport 53 -j DNAT --to-destination 192.168.50.3:53
-A PREROUTING -p tcp --dport 53 -j DNAT --to-destination 192.168.50.3:53
-A POSTROUTING -j MASQUERADE

COMMIT
```

Paso 5. Configurar Cliente (192.168.50.2)

Editar `/etc/resolv.conf` o `/etc/systemd/resolved.conf` y poner:

```bash
nameserver 192.168.50.5
```

Paso 6. Probar

En el cliente:

```bash
dig @192.168.50.5 www.servicios.com
nslookup www.servicios.com 192.168.50.5
```

# Tercer Punto

# Configuración de DNS sobre TLS (DoT)


---


### **Paso 1: Configuración del Archivo resolved.conf**
```bash
sudo nano /etc/systemd/resolved.conf
```

**Contenido configurado:**
```ini
[Resolve]
DNS=8.8.8.8
FallbackDNS=1.1.1.1 9.9.9.9
DNSSEC=yes
DNSOverTLS=yes
```

### **Paso 2: Aplicar la Configuración**
```bash
sudo systemctl restart systemd-resolved
sudo resolvectl flush-caches
```

### **Paso 3: Verificar la Configuración**
```bash
resolvectl status
```
**Salida esperada:**
```
Global
           Protocols: -LLMNR -mDNS +DNSOverTLS DNSSEC=yes/supported
    resolv.conf mode: stub
  Current DNS Server: 8.8.8.8
         DNS Servers: 8.8.8.8
Fallback DNS Servers: 1.1.1.1 9.9.9.9
```

### **Paso 4: Probar Funcionamiento DNS**
```bash
nslookup google.com
nslookup example.com
```


### **Paso 5: Capturar Tráfico DoT**
```bash
# Terminal 1 - Capturar tráfico
sudo tcpdump -i any -n port 853 -w captura_dot.pcap

# Terminal 2 - Generar consultas DNS
nslookup google.com
nslookup cloudflare.com
```
