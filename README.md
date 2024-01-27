# Redes Privadas Virtuales
## A) VPN de acceso remoto con OpenVPN y certificados x509 (5 puntos)
```
• Uno de los dos equipos (el que actuará como servidor) estará conectado a dos redes     

• Para la autenticación de los extremos se usarán obligatoriamente certificados digitales, que se generarán utilizando openssl y se almacenarán en el directorio /etc/openvpn, junto con  los parámetros Diffie-Helman y el certificado de la propia Autoridad de Certificación.     

• Se utilizarán direcciones de la red 10.99.99.0/24 para las direcciones virtuales de la VPN. La dirección 10.99.99.1 se asignará al servidor VPN.     

• Los ficheros de configuración del servidor y del cliente se crearán en el directorio /etc/openvpn de cada máquina, y se llamarán servidor.conf y cliente.conf respectivamente.     

• Tras el establecimiento de la VPN, la máquina cliente debe ser capaz de acceder a una máquina que esté en la otra red a la que está conectado el servidor.

Documenta el proceso detalladamente.
```


En primer lugar de todo en el servidor VPN deberemos activar el bit de forwarding, mejor si es de manera permanente.
``` bash
root@debian:~# cat /etc/sysctl.conf | grep ip_forward
net.ipv4.ip_forward=1
```


Iniciaremos el PKI (Infrastructure de Clave Pública) de OpenVPN utilizando el comando init-pki en el directorio de easy-rsa es el primer paso para establecer una infraestructura de clave pública para OpenVPN.
Esta acción crea una estructura de directorios y archivos necesarios para la gestión de certificados y claves dentro de nuestro servidor de OpenVPN.

![[Pasted image 20240127182849.png]]

Vamos a generar el certificado y clave de la autoridad certificadora.

``` bash
root@debian:/etc/openvpn/easy-rsa# ./easyrsa build-ca
* Notice:
Using Easy-RSA configuration from: /etc/openvpn/easy-rsa/pki/vars

* Notice:
Using SSL: openssl OpenSSL 3.0.11 19 Sep 2023 (Library: OpenSSL 3.0.11 19 Sep 2023)


Enter New CA Key Passphrase: 
Re-Enter New CA Key Passphrase: 
Using configuration from /etc/openvpn/easy-rsa/pki/96af9db9/temp.f327314e
..+......+.....+.++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:Alexvpn

* Notice:

CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/etc/openvpn/easy-rsa/pki/ca.crt
```

Se ha generado un certificado y la clave privada para la autoridad certificadora.

![[Pasted image 20240127183224.png]]
![[Pasted image 20240127183247.png]]


Ahora generamos el certificado del servidor VPN.
```bash
root@debian:/etc/openvpn/easy-rsa# ./easyrsa build-server-full server nopass
* Notice:
Using Easy-RSA configuration from: /etc/openvpn/easy-rsa/pki/vars

* Notice:
Using SSL: openssl OpenSSL 3.0.11 19 Sep 2023 (Library: OpenSSL 3.0.11 19 Sep 2023)

...+......+.........+.....+...+...+++++++++++++++++++++++++++++++++++++++++++++++....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
-----
* Notice:

Keypair and certificate request completed. Your files are:
req: /etc/openvpn/easy-rsa/pki/reqs/server.req
key: /etc/openvpn/easy-rsa/pki/private/server.key


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a server certificate for 825 days:

subject=
    commonName                = server


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes

Using configuration from /etc/openvpn/easy-rsa/pki/59940daa/temp.86c03b2e
Enter pass phrase for /etc/openvpn/easy-rsa/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'server'
Certificate is to be certified until May  1 17:43:08 2026 GMT (825 days)

Write out database with 1 new entries
Database updated

* Notice:
Certificate created at: /etc/openvpn/easy-rsa/pki/issued/server.crt
```

Podemos ver que se ha generado el certificado y la clave privada del servidor VPN.
![[Pasted image 20240127184624.png]]

![[Pasted image 20240127184649.png]]

Ahora generaremos los parámetros Diffie-Helman

```bash
root@debian:/etc/openvpn/easy-rsa# ./easyrsa gen-dh
* Notice:
Using Easy-RSA configuration from: /etc/openvpn/easy-rsa/pki/vars

* Notice:
Using SSL: openssl OpenSSL 3.0.11 19 Sep 2023 (Library: OpenSSL 3.0.11 19 Sep 2023)

Generating DH parameters, 2048 bit long safe prime
.................................................................................*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*

* Notice:

DH parameters of size 2048 created at /etc/openvpn/easy-rsa/pki/dh.pem
```
Se han generado los parámetros de Diffie-Hellman necesarios para establecer una comunicación segura y establecer una clave secreta compartida entre el servidor OpenVPN y los clientes VPN. 

![[Pasted image 20240127185259.png]]

Ahora generamos para el cliente VPN.
```bash
root@debian:/etc/openvpn/easy-rsa# ./easyrsa build-client-full clientevpn nopass
* Notice:
Using Easy-RSA configuration from: /etc/openvpn/easy-rsa/pki/vars

* Notice:
Using SSL: openssl OpenSSL 3.0.11 19 Sep 2023 (Library: OpenSSL 3.0.11 19 Sep 2023)

......+.........+...+...++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
-----
* Notice:

Keypair and certificate request completed. Your files are:
req: /etc/openvpn/easy-rsa/pki/reqs/clientevpn.req
key: /etc/openvpn/easy-rsa/pki/private/clientevpn.key


You are about to sign the following certificate.
Please check over the details shown below for accuracy. Note that this request
has not been cryptographically verified. Please be sure it came from a trusted
source or that you have verified the request checksum with the sender.

Request subject, to be signed as a client certificate for 825 days:

subject=
    commonName                = clientevpn


Type the word 'yes' to continue, or any other input to abort.
  Confirm request details: yes

Using configuration from /etc/openvpn/easy-rsa/pki/446560eb/temp.a3bdec01
Enter pass phrase for /etc/openvpn/easy-rsa/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'clientevpn'
Certificate is to be certified until May  1 17:55:17 2026 GMT (825 days)

Write out database with 1 new entries
Database updated

* Notice:
Certificate created at: /etc/openvpn/easy-rsa/pki/issued/clientevpn.crt
```

Ya hemos generado el certificado y la clave privada para el cliente VPN.

![[Pasted image 20240127185628.png]]
![[Pasted image 20240127185659.png]]

Ahora deberemos deberemos enviarle al cliente VPN el certificado, la clave privada y el certificado de la autoridad certificadora.

En primer lugar copiaremos los archivos necesarios al directorio home del usuario.

```bash
root@debian:/etc/openvpn/easy-rsa/pki# cp ca.crt /home/debian
root@debian:/etc/openvpn/easy-rsa/pki# cp issued/clientevpn.crt /home/debian
root@debian:/etc/openvpn/easy-rsa/pki# cp private/clientevpn.key /home/debian
root@debian:/etc/openvpn/easy-rsa/pki# cd /home/debian
root@debian:/home/debian# ls -l
total 16
-rw------- 1 root root 1188 Jan 27 18:01 ca.crt
-rw------- 1 root root 4485 Jan 27 18:01 clientevpn.crt
-rw------- 1 root root 1704 Jan 27 18:01 clientevpn.key
```

Como podemos ver el propietario de los archivos es root, es por ello que vamos a cambiarlo para poder enviárselo al clienteVPN
```bash
root@debian:/home/debian# chown debian:debian *
root@debian:/home/debian# ls -l
total 16
-rw------- 1 debian debian 1188 Jan 27 18:01 ca.crt
-rw------- 1 debian debian 4485 Jan 27 18:01 clientevpn.crt
-rw------- 1 debian debian 1704 Jan 27 18:01 clientevpn.key
```

Con scp enviaremos los archivos al clienteVPN a su directorio /home/debian
```bash
root@debian:/home/debian# scp * debian@192.168.1.100:/home/debian
The authenticity of host '192.168.1.100 (192.168.1.100)' can't be established.
ED25519 key fingerprint is SHA256:zn2i5rAyilMi1i+Kqb6ys8GhldKuHKYZCDKbD1aXqjQ.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.1.100' (ED25519) to the list of known hosts.
debian@192.168.1.100's password: 
ca.crt                                        100% 1188   652.1KB/s   00:00    
clientevpn.crt                                100% 4485     2.7MB/s   00:00    
clientevpn.key                                100% 1704     1.3MB/s   00:00
```

Como podemos ver desde el cliente vpn ya han sido recibidos los archivos necesarios.
```bash
debian@cliente-vpn-alex:~$ ls -l
total 16
-rw------- 1 debian debian 1188 Jan 27 18:05 ca.crt
-rw------- 1 debian debian 4485 Jan 27 18:05 clientevpn.crt
-rw------- 1 debian debian 1704 Jan 27 18:05 clientevpn.key
```


```bash
root@debian:/etc/openvpn# cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server/servidor.conf
```


Para la configuración del servidor VPN copiaremos la plantilla de la siguiente ruta y la guardaremos en nuestro directorio de trabajo.
```bash
root@debian:/etc/openvpn# cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server/servidor.conf
```

En el fichero de configuración indicaremos donde se encuentran, la autoridad certificadora, el certificado del servidor, la clave privada y los parámetros de Diffie-Hellman.
Aquí vemos como queda, he eliminado todas las lineas de comentarios informativos.

```bash
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

push "route 192.168.2.0 255.255.255.0"   # Empuja la ruta hacia la subred 192.168.2.0/24 a los clientes VPN. Para permitir conexión remota con los clientes en esa red.

keepalive 10 120          # Enviará paquetes de control para mantener activo.

cipher AES-256-CBC        # Utiliza el cifrado AES-256-CBC para la comunicación.

persist-key               # La clave de cifrado entre conexiones será persistente.
persist-tun               # La interfaz de red virtual entre conexiones será persistente.

status /var/log/openvpn/openvpn-status.log   # Ruta para el archivo de registro de estado del servidor OpenVPN.

verb 3                    # Nivel de verbosidad de los registros.
explicit-exit-notify 1    # Notifica a los clientes cuando el servidor se apaga.

```

Iniciamos el servidor y comprobamos que está activo con los siguientes comandos:
```bash
root@debian:/etc/openvpn/server# systemctl enable --now openvpn-server@servidor
root@debian:/etc/openvpn/server# systemctl status openvpn-server@servidor

```
![[Pasted image 20240127203346.png]]

Desde el clienteVPN, deberemos tener instalado openvpn al igual que en el servidor, y ahora moveremos los archivos a su directorio y le cambiaremos el propietario.

```bash
root@cliente-vpn-alex:/home/debian# ls -l
total 16
-rw------- 1 debian debian 1188 Jan 27 18:05 ca.crt
-rw------- 1 debian debian 4485 Jan 27 18:05 clientevpn.crt
-rw------- 1 debian debian 1704 Jan 27 18:05 clientevpn.key
root@cliente-vpn-alex:/home/debian# mv * /etc/openvpn/client/
root@cliente-vpn-alex:/home/debian# cd /etc/openvpn/client/
root@cliente-vpn-alex:/etc/openvpn/client# ls -l
total 16
-rw------- 1 debian debian 1188 Jan 27 18:05 ca.crt
-rw------- 1 debian debian 4485 Jan 27 18:05 clientevpn.crt
-rw------- 1 debian debian 1704 Jan 27 18:05 clientevpn.key
root@cliente-vpn-alex:/etc/openvpn/client# chown root:root *
root@cliente-vpn-alex:/etc/openvpn/client# ls -l
total 16
-rw------- 1 root root 1188 Jan 27 18:05 ca.crt
-rw------- 1 root root 4485 Jan 27 18:05 clientevpn.crt
-rw------- 1 root root 1704 Jan 27 18:05 clientevpn.key
```


```bash
root@cliente-vpn-alex:/etc/openvpn/client# cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf cliente.conf
root@cliente-vpn-alex:/etc/openvpn/client# ls -l
total 20
-rw------- 1 root root 1188 Jan 27 18:05 ca.crt
-rw-r--r-- 1 root root 3591 Jan 27 19:43 cliente.conf
-rw------- 1 root root 4485 Jan 27 18:05 clientevpn.crt
-rw------- 1 root root 1704 Jan 27 18:05 clientevpn.key
```


Modificaremos la configuración para el cliente (he eliminado todas las lineas de comentarios): 

```bash
root@cliente-vpn-alex:/etc/openvpn/client# cat cliente.conf 
client                   # Configuración para un cliente OpenVPN.
dev tun                  # Utiliza el dispositivo TUN para la interfaz de red virtual.
proto udp                # Usa el protocolo UDP para la comunicación.

remote 192.168.1.100 1194   # Dirección IP y el puerto del servidor OpenVPN al que se conectará el cliente.
resolv-retry infinite   # Intenta resolver la dirección del servidor de manera infinita.
nobind                   # No se unirá a una dirección o puerto específico.

persist-key              # Clave de cifrado entre conexiones persistente.
persist-tun              # Interfaz de red virtual entre conexiones persistente.

ca /etc/openvpn/client/ca.crt         # Ruta del certificado de la autoridad de certificación (CA).
cert /etc/openvpn/client/clientevpn.crt   # Ruta del certificado del cliente VPN.
key /etc/openvpn/client/clientevpn.key   # Ruta de la clave privada del cliente VPN.

remote-cert-tls server  # Obligatorio que el certificado del servidor sea válido y esté firmado por la CA.
cipher AES-256-CBC       # Utiliza el cifrado AES-256-CBC para la comunicación.
verb 3                   # Nivel de verbosidad de los registros.
```


Entonces activamos el servicio del cliente y comprobamos que se encuentre activo:

```bash
root@cliente-vpn-alex:/etc/openvpn/client# systemctl enable --now openvpn-client@cliente
root@cliente-vpn-alex:/etc/openvpn/client# systemctl status openvpn-client@cliente
```

![[Pasted image 20240127210006.png]]


Finalmente desde el cliente interno a la VPN estableceremos una ruta por defecto hacia la dirección IP del servidor VPN
```bash
root@cliente-alex:~# ip route add default via 192.168.2.200
root@cliente-alex:~# ip r
default via 192.168.2.200 dev ens4 
192.168.2.0/24 dev ens4 proto kernel scope link src 192.168.2.100 
```

Ahora ya vamos a proceder a realizar pruebas para verificar el correcto funcionamiento de la VPN.

- Realizamos un ping al cliente interno de VPN y llega con éxito!
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


- Realizamos un traceroute para verificar que usa la VPN con éxito!
```bash
debian@cliente-vpn-alex:~$ traceroute 192.168.2.100
traceroute to 192.168.2.100 (192.168.2.100), 30 hops max, 60 byte packets
 1  10.99.99.1 (10.99.99.1)  2.708 ms  3.145 ms  3.229 ms
 2  192.168.2.100 (192.168.2.100)  3.588 ms  3.368 ms  3.465 ms
```

- Realizamos una captura de Wireshark del trafico entre ClienteVPN-ServidorVPN y ServidorVPN-ClienteInterno, podemos ver que llega al Cliente interno a través de la dirección de la VPN.
![[Pasted image 20240127214017.png]]

