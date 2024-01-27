VPN de acceso remoto con OpenVPN y certificados x509

Activar bit forwarding:
``` bash
root@debian:~# cat /etc/sysctl.conf | grep ip_forward
net.ipv4.ip_forward=1
```

Copiar configuracion de /usr/share/easy-rsa a /etc/openvpn:
```bash
sudo cp -r /usr/share/easy-rsa /etc/openvpn
```
Iniciamos PKI
``` bash
./easyrsa init-pki
```
Generamos certificado y clave de CA
```
./easyrsa build-ca
```
Generamos certificado y clave de servidor VPN
```
./easyrsa build-server-full server nopass
```
Generar parametros Diffie-Helman
```
./easyrsa gen-dh
```
Generamos certificado y clave para clienteVPN
```
./easyrsa build-client-full clientevpn nopass
```
Para enviar certificado, clave y ca al cliente:
- copiamos a directorio normal
```
cp ca.crt /home/debian
cp issued/clientevpn.crt /home/debian
cp issued/clientevpn.crt /home/debian
```
- cambamos propietario que no sea root:
```
chown debian:debian *
```
- enviamos con scp
```
scp * debian@192.168.1.100:/home/debian
```

Configuracion VPN
- copiar plantilla
```
cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server/servidor.conf
```

- contenido de configuracion:
```
root@debian:/etc/openvpn/server# cat servidor.conf 
port 1194                 # Escucha en el puerto UDP 1194 para conexiones de clientes VPN.
proto udp                 # Protocolo UDP para la comunicación.
dev tun                   # Usa el dispositivo TUN para la interfaz de red virtual.

ca /etc/openvpn/easy-rsa/pki/ca.crt   # Ruta del certificado de la autoridad de certificación (CA).
cert /etc/openvpn/easy-rsa/pki/issued/server.crt  # Ruta del certificado del servidor.
key /etc/openvpn/easy-rsa/pki/private/server.key  # Ruta de la clave privada del servidor.
dh /etc/openvpn/easy-rsa/pki/dh.pem   # Ruta de los parámetros de Diffie-Hellman.

topology subnet           # Configura la topología de red como subnet.

server 10.99.99.0 255.255.255.0   # Configura el servidor VPN para la red 10.99.99.0/24.
ifconfig-pool-persist /var/log/openvpn/ipp.txt   # Asignaciones de direcciones IP persistentes.
tls-server #Rol de servidor

comp-lzo #Activar la compresión LZO
push "route 192.168.2.0 255.255.255.0"   # Empuja la ruta hacia la subred 192.168.2.0/24 a los clientes VPN. Para permitir conexión remota con los clientes en esa red.

keepalive 10 120          # Enviará paquetes de control para mantener activo.

cipher AES-256-CBC        # Utiliza el cifrado AES-256-CBC para la comunicación.

persist-key               # La clave de cifrado entre conexiones será persistente.
persist-tun               # La interfaz de red virtual entre conexiones será persistente.

status /var/log/openvpn/openvpn-status.log   # Ruta para el archivo de registro de estado del servidor OpenVPN.

verb 3                    # Nivel de verbosidad de los registros.
explicit-exit-notify 1    # Notifica a los clientes cuando el servidor se apaga.


```

- iniciar servidor y comprobar
```
systemctl enable --now openvpn-server@servidor
systemctl status openvpn-server@servidor
```
- reiniciar si fuera necesario
```
systemctl restart --now openvpn-server@servidor
```

Desde CLIENTEVPN
mover archivos a directorio /etc/openvpn/client y cambiar propietario por root
```
mv * /etc/openvpn/client/
chown root:root *
```

configuracion clientevp
- copiamos plantilla
```
cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf cliente.conf
```

- contenido configuracion
```
client
tls-client               #Rol de cliente
dev tun                  # Utiliza el dispositivo TUN para la interfaz de red virtual.
proto udp                # Usa el protocolo UDP para la comunicación.

remote 192.168.1.100 1194   # Dirección IP y el puerto del servidor OpenVPN al que se conectará el cliente.
resolv-retry infinite   # Intenta resolver la dirección del servidor de manera infinita.
nobind                   # No se unirá a una dirección o puerto específico.

persist-key              # Clave de cifrado entre conexiones persistente.
persist-tun              # Interfaz de red virtual entre conexiones persistente.
comp-lzo #Activar la compresión LZO
ca /etc/openvpn/client/ca.crt         # Ruta del certificado de la autoridad de certificación (CA).
cert /etc/openvpn/client/clientevpn.crt   # Ruta del certificado del cliente VPN.
key /etc/openvpn/client/clientevpn.key   # Ruta de la clave privada del cliente VPN.

remote-cert-tls server  # Obligatorio que el certificado del servidor sea válido y esté firmado por la CA.
cipher AES-256-CBC       # Utiliza el cifrado AES-256-CBC para la comunicación.
verb 3                   # Nivel de verbosidad de los registros.
```

- activamos y comprobamos, y reinicar si fuera necesario
```
systemctl enable --now openvpn-client@cliente
systemctl status openvpn-client@cliente
systemctl restart --now openvpn-client@cliente
```

DESDE CLIENTE INTERO
establecer ruta por defecto hacia el servidorvpn
ip route add default via 192.168.2.200

PRUEBAS

desde clientevpn
```bash
debian@cliente-vpn-alex:~$ ping 192.168.2.100
PING 192.168.2.100 (192.168.2.100) 56(84) bytes of data.
64 bytes from 192.168.2.100: icmp_seq=1 ttl=63 time=4.23 ms
64 bytes from 192.168.2.100: icmp_seq=2 ttl=63 time=4.63 ms
64 bytes from 192.168.2.100: icmp_seq=3 ttl=63 time=4.64 ms
64 bytes from 192.168.2.100: icmp_seq=4 ttl=63 time=3.87 ms
^C
--- 192.168.2.100 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 3.865/4.340/4.635/0.319 ms
```

```bash
debian@cliente-vpn-alex:~$ traceroute 192.168.2.100
traceroute to 192.168.2.100 (192.168.2.100), 30 hops max, 60 byte packets
 1  10.99.99.1 (10.99.99.1)  2.708 ms  3.145 ms  3.229 ms
 2  192.168.2.100 (192.168.2.100)  3.588 ms  3.368 ms  3.465 ms
```