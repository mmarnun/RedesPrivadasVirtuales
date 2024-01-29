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

### Servidor VPN
Antes de empezar por supuesto debemos tener instalado OpenVPN podremos con:
```bash
sudo apt install openvpn
```

En primer lugar de todo en el servidor VPN deberemos activar el bit de forwarding, mejor si es de manera permanente.
``` bash
root@debian:~# cat /etc/sysctl.conf | grep ip_forward
net.ipv4.ip_forward=1
```

![](imagenes/Pasted%20image%2020240127182659.png)

Creamos nuestro directorio de trabajo copiando la configuracion de /usr/share/easy-rsa a nuestro directorio.
Después iniciaremos el PKI (Infrastructure de Clave Pública) de OpenVPN utilizando el comando init-pki en el directorio de easy-rsa que es el primer paso para establecer una infraestructura de clave pública para OpenVPN.
Esta acción crea una estructura de directorios y archivos necesarios para la gestión de certificados y claves dentro de nuestro servidor de OpenVPN.

![](imagenes/Pasted%20image%2020240127182849.png)

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

![](imagenes/Pasted%20image%2020240127183224.png)


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
![](imagenes/Pasted%20image%2020240127184624.png)


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

![](imagenes/Pasted%20image%2020240127185259.png)

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

![](imagenes/Pasted%20image%2020240127185628.png)
![](imagenes/Pasted%20image%2020240127185659.png)

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

Iniciamos el servidor y comprobamos que está activo con los siguientes comandos:
```bash
root@debian:/etc/openvpn/server# systemctl enable --now openvpn-server@servidor
root@debian:/etc/openvpn/server# systemctl status openvpn-server@servidor

```
![](imagenes/Pasted%20image%2020240127203346.png)

### Cliente VPN
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


Entonces activamos el servicio del cliente y comprobamos que se encuentre activo:

```bash
root@cliente-vpn-alex:/etc/openvpn/client# systemctl enable --now openvpn-client@cliente
root@cliente-vpn-alex:/etc/openvpn/client# systemctl status openvpn-client@cliente
```

![](imagenes/Pasted%20image%2020240127210006.png)

### Cliente Interno
Finalmente desde el cliente interno a la VPN estableceremos una ruta por defecto hacia la dirección IP del servidor VPN
```bash
root@cliente-alex:~# ip route add default via 192.168.2.200
root@cliente-alex:~# ip r
default via 192.168.2.200 dev ens4 
192.168.2.0/24 dev ens4 proto kernel scope link src 192.168.2.100 
```

### Pruebas
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
![](imagenes/Pasted%20image%2020240127214017.png)

## B) VPN sitio a sitio con OpenVPN y certificados x509

Configura una conexión VPN sitio a sitio entre dos equipos del cloud:
```
• Cada equipo estará conectado a dos redes, una de ellas en común 

• Para la autenticación de los extremos se usarán obligatoriamente certificados digitales, que se generarán utilizando openssl y se almacenarán en el directorio /etc/openvpn, junto con con los parámetros Diffie-Helman y el certificado de la propia Autoridad de Certificación. 

• Se utilizarán direcciones de la red 10.99.99.0/24 para las direcciones virtuales de la VPN. 

• Tras el establecimiento de la VPN, una máquina de cada red detrás de cada servidor VPN debe ser capaz de acceder a una máquina del otro extremo. 

Documenta el proceso detalladamente.
```


### Servidor VPN 2
Antes de empezar por supuesto debemos tener instalado OpenVPN podremos con:
```bash
sudo apt install openvpn
```

En primer lugar de todo en el servidor VPN deberemos activar el bit de forwarding, mejor si es de manera permanente.
``` bash
root@servidor-vpn2-alex:~# cat /etc/sysctl.conf | grep ip_forward
net.ipv4.ip_forward=1
```

Creamos la infraestructura de PKI como hemos hecho anteriormente.
```bash
root@servidor-vpn2-alex:~# cp -r /usr/share/easy-rsa /etc/openvpn
root@servidor-vpn2-alex:~# cd /etc/openvpn/easy-rsa
root@servidor-vpn2-alex:/etc/openvpn/easy-rsa# ./easyrsa init-pki
* Notice:

  init-pki complete; you may now create a CA or requests.

  Your newly created PKI dir is:
  * /etc/openvpn/easy-rsa/pki

* Notice:
  IMPORTANT: Easy-RSA 'vars' file has now been moved to your PKI above.
```

Generamos el certificado y la clave privada de la autoridad certificadora.
```bash
root@servidor-vpn2-alex:/etc/openvpn/easy-rsa# ./easyrsa build-ca
* Notice:
Using Easy-RSA configuration from: /etc/openvpn/easy-rsa/pki/vars

* Notice:
Using SSL: openssl OpenSSL 3.0.11 19 Sep 2023 (Library: OpenSSL 3.0.11 19 Sep 2023)


Enter New CA Key Passphrase: 
Re-Enter New CA Key Passphrase: 
Using configuration from /etc/openvpn/easy-rsa/pki/0b2f288c/temp.17805c38
......+.+..+....+......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*..+.........+.......+........+.+..+...+.+.........+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*....+.........+..+...+...+.......+......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
............+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*......+.+...+...........+....+.....+.+.................+..........+..+.+..............+.+.....+......+......+...+.+......+.........+.........+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
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

Ahora generamos el certificado y la clave privada del servidor VPN:
```bash
root@servidor-vpn2-alex:/etc/openvpn/easy-rsa# ./easyrsa build-server-full server nopass
* Notice:
Using Easy-RSA configuration from: /etc/openvpn/easy-rsa/pki/vars

* Notice:
Using SSL: openssl OpenSSL 3.0.11 19 Sep 2023 (Library: OpenSSL 3.0.11 19 Sep 2023)

....................+..........+...+......+............+..+......+.+.............+++++++++++++++++
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

Using configuration from /etc/openvpn/easy-rsa/pki/12f8771a/temp.6307258a
Enter pass phrase for /etc/openvpn/easy-rsa/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'server'
Certificate is to be certified until May  1 22:41:10 2026 GMT (825 days)

Write out database with 1 new entries
Database updated

* Notice:
Certificate created at: /etc/openvpn/easy-rsa/pki/issued/server.crt
```

Generamos los parámetros de Diffie-Hellman.
```bash
root@servidor-vpn2-alex:/etc/openvpn/easy-rsa# ./easyrsa gen-dh
* Notice:
Using Easy-RSA configuration from: /etc/openvpn/easy-rsa/pki/vars

* Notice:
Using SSL: openssl OpenSSL 3.0.11 19 Sep 2023 (Library: OpenSSL 3.0.11 19 Sep 2023)

Generating DH parameters, 2048 bit long safe prime
.................................................................................+*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*++*

* Notice:

DH parameters of size 2048 created at /etc/openvpn/easy-rsa/pki/dh.pem
```

Creamos el certificado y la clave privada del cliente VPN:
```bash
root@servidor-vpn2-alex:/etc/openvpn/easy-rsa# ./easyrsa build-client-full clientevpn nopass
* Notice:
Using Easy-RSA configuration from: /etc/openvpn/easy-rsa/pki/vars

* Notice:
Using SSL: openssl OpenSSL 3.0.11 19 Sep 2023 (Library: OpenSSL 3.0.11 19 Sep 2023)

....+..+.+..+...+.......+..+..........+..+...............+.+..+.+..+.......++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
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

Using configuration from /etc/openvpn/easy-rsa/pki/02743a30/temp.8b5c807b
Enter pass phrase for /etc/openvpn/easy-rsa/pki/private/ca.key:
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'clientevpn'
Certificate is to be certified until May  1 22:43:59 2026 GMT (825 days)

Write out database with 1 new entries
Database updated

* Notice:
Certificate created at: /etc/openvpn/easy-rsa/pki/issued/clientevpn.crt
```

Movemos los archivos necesarios para el cliente VPN a un directorio mas accesible y cambiamos propietario.
```bash
root@servidor-vpn2-alex:/etc/openvpn/easy-rsa/pki# cp ca.crt /home/debian
root@servidor-vpn2-alex:/etc/openvpn/easy-rsa/pki# cp issued/clientevpn.crt /home/debian
root@servidor-vpn2-alex:/etc/openvpn/easy-rsa/pki# cp private/clientevpn.key /home/debian
root@servidor-vpn2-alex:/etc/openvpn/easy-rsa/pki# cd /home/debian
root@servidor-vpn2-alex:/home/debian# chown debian:debian *
root@servidor-vpn2-alex:/home/debian# ls -l
total 16
-rw------- 1 debian debian 1188 Jan 27 22:45 ca.crt
-rw------- 1 debian debian 4481 Jan 27 22:45 clientevpn.crt
-rw------- 1 debian debian 1704 Jan 27 22:45 clientevpn.key
```

Y se los enviaremos con scp.
```bash
root@servidor-vpn2-alex:/home/debian# scp * debian@80.0.0.2:/home/debian
The authenticity of host '80.0.0.2 (80.0.0.2)' can't be established.
ED25519 key fingerprint is SHA256:zn2i5rAyilMi1i+Kqb6ys8GhldKuHKYZCDKbD1aXqjQ.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '80.0.0.2' (ED25519) to the list of known hosts.
debian@80.0.0.2's password: 
ca.crt                                        100% 1188   556.9KB/s   00:00    
clientevpn.crt                                100% 4481     1.9MB/s   00:00    
clientevpn.key                                100% 1704     1.0MB/s   00:00 
```

Para la configuración copiaremos la plantilla.
```bash
root@servidor-vpn2-alex:~# cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server/servidor-vpn2.conf
```

En la configuración podemos ver algunas pequeñas diferencias a las anteriores:
```bash
root@servidor-vpn2-alex:/etc/openvpn# cat server/servidor-vpn2.conf 
port 1194
proto udp
dev tun


ca /etc/openvpn/easy-rsa/pki/ca.crt
cert /etc/openvpn/easy-rsa/pki/issued/server.crt
key /etc/openvpn/easy-rsa/pki/private/server.key
dh /etc/openvpn/easy-rsa/pki/dh.pem
tls-server
comp-lzo


ifconfig 10.99.99.1 10.99.99.2 # Establece la dirección IP del extremo del servidor VPN como 10.99.99.1 y la dirección IP del extremo del cliente como 10.99.99.2.
route 192.168.1.0 255.255.255.0 # Ruta estática para enrutar el tráfico destinado a la subred 192.168.1.0/24 a través del túnel VPN.

keepalive 10 120
cipher AES-256-CBC
persist-key
persist-tun
status /var/log/openvpn/openvpn-status.log
verb 3
explicit-exit-notify 1
```

Una vez configurado activamos el servicio y comprobamos que esté activo y funcionando.
```bash
root@servidor-vpn2-alex:/etc/openvpn# systemctl enable --now openvpn-server@servidor-vpn2
root@servidor-vpn2-alex:/etc/openvpn# systemctl status openvpn-server@servidor-vpn2
```

### Servidor VPN  1
Esta máquina actuará como cliente VPN realmente para la configuración, pero como estamos configurándolo para que sea sitio a sitio, también le hemos llamado servidor VPN.

Al igual que el anterior debemos activar el bit de forwarding.
```bash
root@servidor-vpn1-alex:~# cat /etc/sysctl.conf | grep ip_forward
net.ipv4.ip_forward=1
```

Antes desde el servidor VPN 2 nos habíamos enviado mediante scp, los archivos necesarios para el cliente.
Entonces los moveremos a su directorio correspondiente y le cambiaremos el propietario a root.
```bash
root@servidor-vpn1-alex:/home/debian# ls -l
total 16
-rw------- 1 debian debian 1188 Jan 27 22:46 ca.crt
-rw------- 1 debian debian 4481 Jan 27 22:46 clientevpn.crt
-rw------- 1 debian debian 1704 Jan 27 22:46 clientevpn.key
root@servidor-vpn1-alex:/home/debian# mv * /etc/openvpn/server
root@servidor-vpn1-alex:/home/debian# cd /etc/openvpn/server/
root@servidor-vpn1-alex:/etc/openvpn/server# chown root:root *
root@servidor-vpn1-alex:/etc/openvpn/server# ls -l
total 16
-rw------- 1 root root 1188 Jan 27 22:46 ca.crt
-rw------- 1 root root 4481 Jan 27 22:46 clientevpn.crt
-rw------- 1 root root 1704 Jan 27 22:46 clientevpn.key
```


Pasaremos ahora a la configuración.
```bash
root@servidor-vpn1-alex:/etc/openvpn/server# cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf servidor-vpn1.conf
```

Al igual que en la del anterior servidor VPN hay algunos cambios.
```bash
root@servidor-vpn1-alex:/etc/openvpn/server# cat servidor-vpn1.conf 
tls-client
dev tun

remote 80.0.0.1 1194 # Dirección del servidor VPN
ifconfig 10.99.99.2 10.99.99.1 #Establece la dirección IP de los extremos del servidor VPN.
route 192.168.2.0 255.255.255.0 # Ruta estática para enrutar el tráfico destinado a la subred 192.168.2.0/24 a través del túnel VPN.

comp-lzo

ca /etc/openvpn/server/ca.crt
cert /etc/openvpn/server/clientevpn.crt
key /etc/openvpn/server/clientevpn.key

persist-key
persist-tun
cipher AES-256-CBC
keepalive 10 120
log /var/log/openvpn.log
verb 3
```

Una vez configurado activamos el servicio y comprobamos que esté activo y funcionando.
```bash
root@servidor-vpn1-alex:/etc/openvpn/server# systemctl enable --now openvpn-server@servidor-vpn1
root@servidor-vpn1-alex:/etc/openvpn/server# systemctl status openvpn-server@servidor-vpn1
```


### Cliente Interno 2
En ambos clientes deberemos de establecer una ruta por defecto hacia el servidor VPN.
```bash
debian@cliente-int2-alex:~$ sudo ip route add default via 192.168.2.200
debian@cliente-int2-alex:~$ ip r
default via 192.168.2.200 dev ens4 
192.168.2.0/24 dev ens4 proto kernel scope link src 192.168.2.100 
```
### Cliente Interno 1
```bash
debian@cliente-int1-alex:~$ sudo ip route add default via 192.168.1.200
debian@cliente-int1-alex:~$ ip r
default via 192.168.1.200 dev ens4 
192.168.1.0/24 dev ens4 proto kernel scope link src 192.168.1
```
### Pruebas
Ahora ya vamos a proceder a realizar pruebas para verificar el correcto funcionamiento de la VPN.

- Ping desde el Cliente Interno 1 hacia el Cliente Interno 2
```bash
debian@cliente-int1-alex:~$ ping 192.168.2.100
PING 192.168.2.100 (192.168.2.100) 56(84) bytes of data.
64 bytes from 192.168.2.100: icmp_seq=1 ttl=62 time=5.43 ms
64 bytes from 192.168.2.100: icmp_seq=2 ttl=62 time=5.56 ms
64 bytes from 192.168.2.100: icmp_seq=3 ttl=62 time=5.97 ms
64 bytes from 192.168.2.100: icmp_seq=4 ttl=62 time=2.24 ms
^C
--- 192.168.2.100 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 2.242/4.801/5.974/1.491 ms
```

- Ping desde el Cliente Interno 2 hacia el Cliente Interno 1
```bash
debian@cliente-int2-alex:~$ ping 192.168.1.100
PING 192.168.1.100 (192.168.1.100) 56(84) bytes of data.
64 bytes from 192.168.1.100: icmp_seq=1 ttl=62 time=5.39 ms
64 bytes from 192.168.1.100: icmp_seq=2 ttl=62 time=5.39 ms
64 bytes from 192.168.1.100: icmp_seq=3 ttl=62 time=6.32 ms
64 bytes from 192.168.1.100: icmp_seq=4 ttl=62 time=5.95 ms
^C
--- 192.168.1.100 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3006ms
rtt min/avg/max/mdev = 5.388/5.761/6.315/0.392 ms
```

- Realizaremos traceroute desde el Cliente Interno 1 hacia el Cliente Interno 2
```bash
debian@cliente-int1-alex:~$ traceroute 192.168.2.100
traceroute to 192.168.2.100 (192.168.2.100), 30 hops max, 60 byte packets
 1  192.168.1.200 (192.168.1.200)  2.757 ms  2.714 ms  2.692 ms
 2  10.99.99.1 (10.99.99.1)  3.003 ms  2.995 ms  2.989 ms
 3  192.168.2.100 (192.168.2.100)  5.131 ms  5.110 ms  5.089 ms
```

- Realizaremos traceroute desde el Cliente Interno 2 hacia el Cliente Interno 1
```
debian@cliente-int2-alex:~$ traceroute 192.168.1.100
traceroute to 192.168.1.100 (192.168.1.100), 30 hops max, 60 byte packets
 1  192.168.2.200 (192.168.2.200)  1.383 ms  1.320 ms  1.246 ms
 2  10.99.99.2 (10.99.99.2)  7.508 ms  7.336 ms  7.213 ms
 3  192.168.1.100 (192.168.1.100)  7.065 ms  6.961 ms  6.874 ms
```

- Realizamos una captura de Wireshark del trafico que pasa entre los servidores VPN al realizar ping al Cliente Interno 2 cuando realizamos un ping desde el Cliente Interno 1, podemos ver que los paquetes viajan por la red de "Internet" simulado mediante el protocolo de OpenVPN
![](imagenes/Pasted%20image%2020240128115448.png)


## C) VPN de acceso remoto con WireGuard

Monta una VPN de acceso remoto usando Wireguard. Intenta probarla con clientes Windows, Linux y Android. Documenta el proceso adecuadamente y compáralo con el del apartado A.

### Servidor VPN
Antes de empezar por supuesto debemos tener instalado OpenVPN podremos con:
```bash
sudo apt install wireguard
```

Activaremos el bit de forwarding.
```bash
root@servidor-vpn-alex:~# sysctl -p
net.ipv4.ip_forward = 1
```

Vamos a generar las claves publica y privada con este comando generaremos un par de claves para WireGuard y las guardará en una ruta concreta. 
```bash
root@servidor-vpn-alex:/etc/wireguard# wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key

root@servidor-vpn-alex:/etc/wireguard# cat server_public.key 
OknH5XPIshiFUBLzylPdD5xyuXA4mCi9uWUz2rmSoyo=

root@servidor-vpn-alex:/etc/wireguard# cat server_private.key 
GIGe2xa9X6p8Xr9EwBu/COTrzMNkZLBX0DpjtCk2+Vs=
```

Ahora crearemos un fichero de configuración donde incluimos ya la clave publica del cliente que explico como la generamos más adelante:
```bash
root@servidor-vpn-alex:/etc/wireguard# cat wg0.conf 
[Interface]
Address = 10.99.99.1/24 # Direccion IP y mascara de la red virtual
ListenPort = 51820 # Puerto donde WireGuard escuchará las conexiones
PrivateKey = GIGe2xa9X6p8Xr9EwBu/COTrzMNkZLBX0DpjtCk2+Vs=
PreUp = sysctl -w net.ipv4.ip_forward=1 # Activara el bir de forwarding

# Clave publica del cliente linux
[Peer]
PublicKey = M0jW9c33zYn3flZItu6lClZ098GQuaYrmhveLkMrZwA=
AllowedIPs = 10.99.99.2/32 	# Direcciones IP permitidas
Endpoint = 192.168.1.200:51820 # Extremo del tunel
```

Le cambiaremos los permisos por solo lectura.
```bash
root@servidor-vpn-alex:/etc/wireguard# chmod 600 . -R
root@servidor-vpn-alex:/etc/wireguard# ls -l
total 12
-rw------- 1 root root  45 Jan 28 18:10 server_private.key
-rw------- 1 root root  45 Jan 28 18:10 server_public.key
-rw------- 1 root root 400 Jan 28 18:49 wg0.conf
```

Activaremos la configuración con el siguiente comando y ha configurado una interfaz WireGuard según el archivo configurado.
```bash
root@servidor-vpn-alex:/etc/wireguard# wg-quick up /etc/wireguard/wg0.conf 
[#] sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
[#] ip link add wg0 type wireguard
[ 3269.035432] wireguard: WireGuard 1.0.0 loaded. See www.wireguard.com for information.
[ 3269.036939] wireguard: Copyright (C) 2015-2019 Jason A. Donenfeld <Jason@zx2c4.com>. All Rights Reserved.
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.99.99.1/24 dev wg0
[#] ip link set mtu 1420 up dev wg0
```

En caso de ser necesaria después de incluir la clave publica sino lo hubiéramos hecho "reiniciamos" con los siguientes comandos:
```bash
root@servidor-vpn-alex:/etc/wireguard# wg-quick down /etc/wireguard/wg0.conf 
[#] ip link delete dev wg0
root@servidor-vpn-alex:/etc/wireguard# wg-quick up /etc/wireguard/wg0.conf 
[#] sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.99.99.1/24 dev wg0
[#] ip link set mtu 1420 up dev wg0
```

Podemos ver que se ha creado la interfaz correctamente:
```
root@servidor-vpn-alex:/etc/wireguard# ip a
...
4: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
    link/none 
    inet 10.99.99.1/24 scope global wg0
       valid_lft forever preferred_lft foreve
```

Y asimismo la ruta
```
root@servidor-vpn-alex:/etc/wireguard# ip r
10.99.99.0/24 dev wg0 proto kernel scope link src 10.99.99.1 
```
### Cliente VPN Linux
También debemos tener instalado WireGuard por supuesto.
Crearemos al igual que en el servidor un par de claves de WireGuard y las almacenaremos en /etc/wireguard.
```bash
root@cliente-vpn-alex:~# wg genkey | sudo tee /etc/wireguard/client_private.key | wg pubkey | sudo tee /etc/wireguard/client_public.key

root@cliente-vpn-alex:/etc/wireguard# cat client_private.key 
aN5jy3M8rt+DteffYHgqa6RwmLtujsNb7UlA/0PS9nQ=

root@cliente-vpn-alex:/etc/wireguard# cat client_public.key 
M0jW9c33zYn3flZItu6lClZ098GQuaYrmhveLkMrZwA=
```

Crearemos un archivo de configuración:
```bash
root@cliente-vpn-alex:/etc/wireguard# cat wg-client0.conf 
[Interface]
Address = 10.99.99.2/24 # Direccion IP y mascara de la red virtual
PrivateKey = aN5jy3M8rt+DteffYHgqa6RwmLtujsNb7UlA/0PS9nQ=

[Peer]
PublicKey = OknH5XPIshiFUBLzylPdD5xyuXA4mCi9uWUz2rmSoyo=
AllowedIPs = 10.99.99.1/32, 192.168.2.0/24 # Todas las direcciones IP son permitidas al tunel.
Endpoint = 192.168.1.100:51820 # Direccion IP y puerto del servidor WireGuard al que se conectara
PersistentKeepalive = 25 # Intervalo de tiempo en segundos para los paquetes keepalive para la inactividad.
```

Le cambiamos los permisos pues son ficheros delicados.
```
root@cliente-vpn-alex:/etc/wireguard# chmod 600 . -R
root@cliente-vpn-alex:/etc/wireguard# ls -l
total 12
-rw------- 1 root root  45 Jan 28 18:57 client_private.key
-rw------- 1 root root  45 Jan 28 18:57 client_public.key
-rw------- 1 root root 483 Jan 28 19:16 wg-client0.conf
```

Y activaremos la configuración con el siguiente comando:
```bash
root@cliente-vpn-alex:/etc/wireguard# wg-quick up /etc/wireguard/wg-client0.conf 
[#] ip link add wg-client0 type wireguard
[ 4796.196799] wireguard: WireGuard 1.0.0 loaded. See www.wireguard.com for information.
[ 4796.198083] wireguard: Copyright (C) 2015-2019 Jason A. Donenfeld <Jason@zx2c4.com>. All Rights Reserved.
[#] wg setconf wg-client0 /dev/fd/63
[#] ip -4 address add 10.99.99.2/24 dev wg-client0
[#] ip link set mtu 1420 up dev wg-client0
```

En caso de necesitar reiniciarlo al introducirle la clave pública del servidor reiniciaremos con estos comandos:
```bash
root@cliente-vpn-alex:/etc/wireguard# wg-quick down /etc/wireguard/wg-client0.conf
[#] ip -4 rule delete table 51820
[#] ip -4 rule delete table main suppress_prefixlength 0
[#] ip link delete dev wg-client0
[#] iptables-restore -n
root@cliente-vpn-alex:/etc/wireguard# wg-quick up /etc/wireguard/wg-client0.conf[#] ip link add wg-client0 type wireguard
[#] wg setconf wg-client0 /dev/fd/63
[#] ip -4 address add 10.99.99.2/24 dev wg-client0
[#] ip link set mtu 65456 up dev wg-client0
[#] wg set wg-client0 fwmark 51820
[#] ip -4 route add 0.0.0.0/0 dev wg-client0 table 51820
[#] ip -4 rule add not fwmark 51820 table 51820
[#] ip -4 rule add table main suppress_prefixlength 0
[#] sysctl -q net.ipv4.conf.all.src_valid_mark=1
[#] iptables-restore -n
```

Podemos ver que ha creado la interfaz de WireGuard:
```bash
root@cliente-vpn-alex:/etc/wireguard# ip a
...
3: wg-client0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
    link/none 
    inet 10.99.99.2/24 scope global wg-client0
       valid_lft forever preferred_lft forever
```

Asimismo la ruta:
```bash
root@cliente-vpn-alex:/etc/wireguard# ip r
10.99.99.0/24 dev wg-client0 proto kernel scope link src 10.99.99.2 
```

Aunque debemos agregarle una por defecto hacia la dirección IP de la VPN:
```bash
debian@cliente-vpn-alex:~$ sudo ip route add default via 10.99.99.1
debian@cliente-vpn-alex:~$ ip r
default via 10.99.99.1 dev wg-client0 
10.99.99.0/24 dev wg-client0 proto kernel scope link src 10.99.99.2 
```

### Cliente Interno
Al cliente interno le estableceremos la ruta por defecto hacia el servidor VPN.
```bash
debian@cliente-int-alex:~$ sudo ip route add default via 192.168.2.100
debian@cliente-int-alex:~$ ip r
default via 192.168.2.100 dev ens4 
```

### Pruebas

- Ping desde el cliente VPN hacia el cliente Interno.
```bash
debian@cliente-vpn-alex:~$ ping 192.168.2.200
PING 192.168.2.200 (192.168.2.200) 56(84) bytes of data.
64 bytes from 192.168.2.200: icmp_seq=1 ttl=63 time=3.85 ms
64 bytes from 192.168.2.200: icmp_seq=2 ttl=63 time=1.51 ms
64 bytes from 192.168.2.200: icmp_seq=3 ttl=63 time=2.05 ms
64 bytes from 192.168.2.200: icmp_seq=4 ttl=63 time=2.49 ms
^C
--- 192.168.2.200 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 1.514/2.476/3.852/0.865 ms
```

- Traceroute desde el cliente VPN hacia el cliente Interno.
```bash 
debian@cliente-vpn-alex:~$ traceroute 192.168.2.200
traceroute to 192.168.2.200 (192.168.2.200), 30 hops max, 60 byte packets
 1  10.99.99.1 (10.99.99.1)  1.576 ms  1.498 ms  1.473 ms
 2  192.168.2.200 (192.168.2.200)  2.380 ms  2.287 ms  2.168 ms
```

- Realizamos el ping capturando el trafico entre el servidor VPN y el cliente interno y podemos ver que llegan los paquetes desde la dirección IP del túnel.

![](imagenes/Pasted%20image%2020240129005352.png)

### Cliente VPN Windows
En primer lugar le colocaremos una dirección IP estática con el siguiente comando:
```c
netsh interface ipv4 set address name="Instancia de Ethernet 0 2" static 192.168.1.210 255.255.255.0 192.168.1.100
```

![](imagenes/Pasted%20image%2020240129011725.png)

A continuación en el cliente VPN de Windows instalaremos WireGuard desde www.wireguard.com/install debemos descargar la versión para Windows, y lo ejecutaremos.

Al ejecutarlo se nos abre una ventana donde nos dirigiremos a añadir un nuevo túnel vacío.

![](imagenes/Pasted%20image%2020240128222451.png)

Una vez le hagamos click se nos abre otra nueva ventana y veremos que ya ha creado el par de claves pública y privada.

![](imagenes/Pasted%20image%2020240128222530.png)


Generamos la configuración como cliente y guardamos.

![](imagenes/Pasted%20image%2020240129010407.png)

### Servidor VPN

```
root@servidor-vpn-alex:/etc/wireguard# cat wg0.conf 
[Interface]
Address = 10.99.99.1/24 # Direccion IP y mascara de la red virtual
ListenPort = 51820 # Puerto donde WireGuard escuchará las conexiones
PrivateKey = GIGe2xa9X6p8Xr9EwBu/COTrzMNkZLBX0DpjtCk2+Vs=
PreUp = sysctl -w net.ipv4.ip_forward=1 # Activara el bir de forwarding

# Clave publica del cliente linux
[Peer]
PublicKey = M0jW9c33zYn3flZItu6lClZ098GQuaYrmhveLkMrZwA=
AllowedIPs = 10.99.99.2/32 	# Direcciones IP permitidas
Endpoint = 192.168.1.200:51820

# Clave publica del cliente windows
[Peer]
PublicKey = IqV7jyvSg5oWs+0DPsfp92RiDYQ0oA9SY4WB6BRcuSU=
AllowedIPs = 10.99.99.3/32
Endpoint = 192.168.1.210:51820
```

Y reiniciamos la interfaz:
```
root@servidor-vpn-alex:/etc/wireguard# wg-quick down /etc/wireguard/wg0.conf
[#] ip link delete dev wg0
root@servidor-vpn-alex:/etc/wireguard# wg-quick up /etc/wireguard/wg0.conf
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.99.99.1/24 dev wg0
[#] ip link set mtu 1420 up dev wg0
```


### Cliente VPN Windows
De nuevo desde el Cliente VPN de Windows activaremos el túnel que hemos creado.

![](imagenes/Pasted%20image%2020240128225420.png)


Podemos ver que también nos ha creado la interfaz del túnel.
![](imagenes/Pasted%20image%2020240129011813.png)

