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

<br>

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
```bash



