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

