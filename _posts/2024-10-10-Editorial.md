---
layout: single
title: Editorial Writeup
excerpt: "Descripcion"
date: 2024-10-10
classes: wide
header: 
 teaser: /assets/images/Editorial/teaserImage.webp
 teaser_home_page: true
categories: 
 - Linux
 - Easy
tags:
 - SSRF
 - sudo abuse
 - web
---

# Reconocimiento:

Realizando un reconocimiento de puertos con **`nmap`** mediante *TCP*:

```bash
nmap -p- --open -vvv -n -sS -Pn --min-rate 5000 10.129.28.76
```

<div style="text-align: center;">
  <img src="/assets/images/Editorial/reconports.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

El resultado obtenido, se evidencian que solo están abiertos los puertos *80* y *22*. Debido a este resultado, considero que seria una buena opción también enumerar puertos mediante por el protocolo *UDP* .

```bash
nmap -p- --open -vvv -n -sS -Pn -sU --min-rate 5000 10.129.28.76
```

El escaneo no arrojó algún indicio de servicio corriendo en el servidor.

Escaneo de vhosts:

```bash
gobuster vhost -u http://editorial.htb/ -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt --append-domain -t 20
```

No se obtuvo información de algún vhosts corriendo en la maquina victima.

Escaneo de directorios: 

```bash
gobuster dir -u http://editorial.htb -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -t 20
```

El resultado del escaneo muestra los siguientes directorios:

<div style="text-align: center;">
  <img src="/assets/images/Editorial/recondir.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

El directorio */upload*  llama la atención, ya que podría ser un vector para encontrar contenido sensible o desencadenar la ejecución de un payload malicioso que carguemos en alguna otra parte de la página. Sin embargo, al revisarlo en el navegador, encontramos que contiene un formulario.

<div style="text-align: center;">
  <img src="/assets/images/Editorial/uploaddir.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

Los formularios son siempre un posible vector de ataque para explotar vulnerabilidades como **XSS, SQLi y SSRF**. En este caso, descartamos un **CSRF** ya que no hay un apartado de inicio de sesión que permita inducir acciones no deseadas a un administrador o usuario con privilegios elevados. Sin embargo, al considerar que el formulario permite cargar archivos e ingresar una URL, los principales vectores de ataque serían la carga de contenido malicioso o un SSRF.

--------------------------------------------------

# Explotaciòn 

Analizando las peticiones mediante BurpSuite, intenté cargar un archivo malicioso para obtener una reverse shell, sin éxito. Sin embargo, al ingresar información en el campo **"Cover URL related to your book or"** y cargar una imagen, probamos un **SSRF**. 

<div style="text-align: center;">
  <img src="/assets/images/Editorial/SSRFPOC1.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

Levantamos un servicio HTTP en nuestra máquina atacante para verificar que la petición proviene desde la IP de la máquina víctima y no desde el cliente.

<div style="text-align: center;">
  <img src="/assets/images/Editorial/SSRFPOC2.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

Confirmamos que es vulnerable a un **SSRF**.

<div style="text-align: center;">
  <iframe src="https://giphy.com/embed/dtC7b5Wz2H8rFZceA6" width="600" height="220" style="pointer-events: none" oncontextmenu="return false;" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/snl-saturday-night-live-season-44-dtC7b5Wz2H8rFZceA6"></a></p>
</div>
<br>

Este es un **SSRF blind/Out-of-band**, ya que no podemos ver directamente el contenido del localhost en la página. Para enumerar los puertos ocultos o detrás de un firewall, utilizamos **'ffuf'**.

Se modifica la petición reemplazando el valor a fuzzear con la palabra **FUZZ**.

<div style="text-align: center;">
  <img src="/assets/images/Editorial/fuff1.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

Generamos un diccionario con el numero total de puertos con el siguiente comando: **seq 1 65535 > ports.txt**, 

```bash
ffuf -w /usr/share/SecLists/Discovery/Infrastructure/common-http-ports.txt -request post.req -u http://editorial.htb/upload-cover -fs 61
```

En la identificación de un puerto abierto, se utiliza la misma metodología que en la explotación de vulnerabilidades como, por ejemplo, **SQLi Blind**, donde el primer cambio en el contenido de la cabecera **Content-Length** en la respuesta puede indicar que en ese número de puerto está corriendo un servicio en el localhost de la máquina víctima.

<div style="text-align: center;">
  <img src="/assets/images/Editorial/fuff2.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

Al apuntar al puerto 5000, encontramos un archivo JSON con varios endpoints de una API corriendo en el localhost.

<div style="text-align: center;">
  <img src="/assets/images/Editorial/endpoints.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

Al enumerar el endpoint llamado *authors*, obtenemos las siguiente credenciales:

<div style="text-align: center;">
  <img src="/assets/images/Editorial/authors.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

-------

# Escalada de privilegios:

Al iniciar sesión por SSH como dev, realizamos una enumeración básica.

**Usuarios con una bash:**

```bash
cat /etc/passwd | grep "/bash"

root:x:0:0:root:/root:/bin/bash
prod:x:1000:1000:Alirio Acosta:/home/prod:/bin/bash
dev:x:1001:1001::/home/dev:/bin/bash

```

**Archivos con SUID de root**

```bash
find / -type f -perm -4000 2>/dev/null

/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/libexec/polkit-agent-helper-1
/usr/bin/chsh
/usr/bin/fusermount3
/usr/bin/sudo
/usr/bin/umount
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/su

```

Revisar los grupos a los que pertenece el usuario dev con el comando (id), así como las tareas cron que se estén ejecutando en el sistema mediante:

Durante el proceso de enumeración, encontramos un directorio llamado **"apps"** en el espacio personal del usuario dev, el cual resulta llamativo. Al ingresar y listar su contenido, a simple vista parece vacío, pero al ejecutar un ls -a para mostrar los archivos ocultos, observamos que el directorio contiene un fichero **.git**. Esto abre la posibilidad de ejecutar comandos de Git y revisar el registro de commits del proyecto.

<div style="text-align: center;">
  <img src="/assets/images/Editorial/commitenum.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

Este fue el commit que automáticamente llamó mi atención, ya que el mensaje descriptivo deja bastante claro que hemos obtenido las credenciales de otro usuario del sistema: el usuario prod.

Al iniciar sesión como el usuario **prod** y realizar la enumeración básica mencionada anteriormente, ejecutamos el comando **"sudo -l"** y evidenciamos que podemos ejecutar el siguiente script de Python como root mediante sudo.

<div style="text-align: center;">
  <img src="/assets/images/Editorial/sudoscript.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

Al validar los permisos que tenemos sobre este fichero, encuentro que es posible leer el contenido del script, lo cual resulta útil para identificar qué hace y evaluar qué posible escalada de privilegios podríamos aprovechar.

<div style="text-align: center;">
  <img src="/assets/images/Editorial/sudoscript1.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

Al darle un vistazo, encontramos que el script permite clonar un repositorio de Git en el directorio actual de trabajo, pero no realiza ningún tipo de validación del input que ingresemos. Esto sugiere de inmediato una potencial inyección de comandos para elevar privilegios.

La inyección de comandos reside en la siguiente línea, donde se utiliza el protocolo **ext**:

<div style="text-align: center;">
  <img src="/assets/images/Editorial/sudoscript2.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

Este protocolo permite ejecutar comandos arbitrarios en el sistema sin ningún tipo de restricción.

Si ejecutamos el script pasando como argumento el comando **"whoami"**, obtenemos el siguiente resultado, lo que evidencia que los comandos se están ejecutando como el usuario root:

<div style="text-align: center;">
  <img src="/assets/images/Editorial/commandinjection.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

Para la escalada de privilegios, opté por cambiar los permisos del fichero /etc/shadow ejecutando el siguiente comando:

```bash
chmod o+rw ../../../etc/shadow
```
Teniendo permisos de escritura en este fichero, podemos sobreescribir la contraseña del usuario root y asignar una nueva. Para generar esta nueva contraseña, usamos el siguiente comando:

```bash
openssl passwd -1 "tooradmin123"
```

<div style="text-align: center;">
  <img src="/assets/images/Editorial/sudoscript2.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

Copiamos el hash generado y lo añadimos al fichero */etc/shadow*, luego guardamos los cambios.

Después, realizamos login como el usuario root mediante el comando:

```bash
su root
```

<div style="text-align: center;">
  <img src="/assets/images/Editorial/root.png" alt="editorial" width="500" oncontextmenu="return false;">
</div>
<br>

<div style="text-align: center;">
  <iframe src="https://giphy.com/embed/RPwrO4b46mOdy" width="480" height="274" style="pointer-events: none" oncontextmenu="return false;" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/rami-malek-RPwrO4b46mOdy">via GIPHY</a></p>
</div>
<br>