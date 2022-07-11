---
layout: single	
title: <span class="htb">Fawn - <span class="starting_point">Starting point</span> - Hack The Box</span>
excerpt: "En este write up veremos la segunda máquina del starting point tier 0, basicamente nos conectaremos por ftp mediante
el login anonymous y nos descargaremos la flag en nuestro equipo."
date: 2022-07-06
classes: wide
header:
  teaser: /assets/images/Fawn/fawn.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.png
categories:
  - Hack The Box
  - Starting point
  - Tier 0
tags: 
  - Nmap
  - FTP Anonymous
---

## DESCRIPCION

En este write up veremos la segunda máquina del starting point tier 0, basicamente nos conectaremos por ftp mediante 
el login anonymous y nos descargaremos la flag en nuestro equipo.

## INDICE

- [Reconocimiento de puertos](#reconocimiento-de-puertos)
- [FTP](#acceder-al-ftp)
- [Respuestas HTB](#respuestas)

## RECONOCIMIENTO DE PUERTOS


Con nmap vemos que puertos estan abiertos

```zsh
nmap --open -p- -T5 -oN Puertos 10.129.250.199
Nmap scan report for 10.129.250.199
Host is up (0.040s latency).
Not shown: 65467 closed tcp ports (conn-refused), 67 filtered tcp ports (no-response)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE
21/tcp open  ftp

```
La única vía potencial a primera vista es mediante el protocolo FTP corriendo en el puerto 21, Lanzamos unos scrips predeterminados para obtener información.

```zsh
nmap -sCV -p 21 -oN Objetivos 10.129.250.199
Nmap scan report for 10.129.250.199
Host is up (0.035s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.10.16.46
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
Service Info: OS: Unix

```
Gracias a este resultado podemos ver que el login por Anonymous está permitido, las credenciales son `anonymous:anonymous`. Desde el propio resultado que nos da nmap se puede observar el contenido 
ubicado en el servidor, viendo `-rw-r--r-- 1 0 0 32 Jun 04  2021 flag.txt`

## ACCEDER AL FTP

Desde la propia terminal nos podemos conectar al ftp, acceder con las credenciales anteriormente mostradas y descargar el archivo `flag.txt` con `get`

```zsh
ftp 10.129.250.199
Connected to 10.129.250.199.
220 (vsFTPd 3.0.3)
Name (10.129.250.199:void4m0n): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> dir
229 Entering Extended Passive Mode (|||14535|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
226 Directory send OK.
ftp> get flag.txt
local: flag.txt remote: flag.txt
229 Entering Extended Passive Mode (|||49236|)
150 Opening BINARY mode data connection for flag.txt (32 bytes).
100% |******************************************************************************************************************************************************************|    32        0.37 KiB/s    00:00 ETA
226 Transfer complete.
32 bytes received in 00:00 (0.18 KiB/s)
ftp> 

```
En este momento ya disponemos de la flag en nuestro equipo.

```zsh
 cat flag.txt
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: flag.txt
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ 035db21c881520061c53e0536e44f815
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
```

## RESPUESTAS

### Pregunta 1

What does the 3-letter acronym FTP stand for?

`File Transfer Protocol`

### Pregunta 2

Which port does the FTP service listen on usually?

`21`

### Pregunta 3

What acronym is used for the secure version of FTP?

`SFTP`

### Pregunta 4

What is the command we can use to send an ICMP echo request to test our connection to the target?

`ping` 

### Pregunta 5

From your scans, what version is FTP running on the target?


`vsftpd 3.0.3`

### Pregunta 6

From your scans, what OS type is running on the target? 

`Unix`

### Pregunta 7

What is the command we need to run in order to display the 'ftp' client help menu?

`ftp -h`

### Pregunta 8

What is username that is used over FTP when you want to log in without having an account?

`anonymous`

### Bandera

Submit root flag

`035db21c881520061c53e0536e44f815 `

