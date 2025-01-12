---
title: eJTPv2 Review - Tips para el Examen
published: 2024-08-28
description: 'En este articulo encontraras todo lo que nesecitas para aprobar el eJPTv2. Entre otras cositas.'
image: './img/ppp.png'
tags: ['Certs']
category: 'Certificaciones'
draft: false 
---

Hace unos dias decidi tomar la certificacion ``eJPTv2``, ya que actualmente me ando preparando para tomar el CPTS y no tenia bien claro como era el ambiente de las certificaciones.

En este articulo voy a dar todos los tips y herramientas que yo use en la certificacion.

# Sobre el examen

- Tiempo limite del examen: 2 dias (48 horas).
- 35 Preguntas Basadas en el Entorno que te asignan.
- Te proporcionan una maquina Kali Linux desde el navegador.
- Tu maquina Atacante no tiene Internet
- No puedes copiar y pegar texto desde fuera de la maquina.

# Mi experiencia con el examen

> Compre el vouncher del examen y lo tome al otro dia, obviamente ya tengo unas buenas bases de Pentesting y buen recorrido, por esto no me hizo falta hacer el curso de INE, ni algun entrenamiento antes de tomar el examen.

Empece el examen y en 8 horas ya habia completado todo, pero decidi entregarlo al otro dia para revisar bien las preguntas y verficar que esten todas bien.

En mi caso me tocaron 11 maquinas en total en toda la red (Incluyendo la Red Interna), habian varias maquinas que solo eran de relleno, asi que no te preocupes(``Te puede tocar otro examen diferente al mio``).

Bueno Tampoco quiero hacer tanto spoiler del examen.

### Mi Opinion sobre el eJPTv2?

El examen no es la gran cosa (no es muy dificil), pero tambien depende de tu conocimiento de Pentesting, la verdad puede ser algo incomodo para una persona que esta empezando en el mundo del Hacking.

Pero con las herramientas, Paginas y Temas que te dare mas abajo, lo apruebas segurisimo.

# Herramientas y Paginas Recomendadas

- nmap
- hydra
- john
- crackmapexec 
- smbmap
- smbclient
- whatweb
- hashcat
- dirb
- wpscan
- Metasploit
- Searchsploit
- enum4linux
- GTFObins - https://gtfobins.github.io/
- CrackStation - https://crackstation.net 
- Hacktricks - https://book.hacktricks.xyz/
- Burpsuite
- xfreerdp
- Wordlist Para brute force a Usuarios -> unix_users
- Wordlist Para brute force a Password -> unix_passwords - rockyou
- Barrido de Ping para descubrimiento de Hostss
```bash
# En linux
for i in {1..254} ;do (ping -c 1 172.16.5.$i | grep "bytes from" &) ;done

# En Powershell
1..254 | % {"172.16.5.$($_): $(Test-Connection -count 1 -comp 172.15.5.$($_) -quiet)"}

# En CMD
for /l %i in (1,1,30) do @ping -n 1 -w 300 172.16.6.%i > nul && echo 172.16.6.%i is up || echo 172.16.5.%i is down
```

# Temas a Aprender

- Pivoting con Metasploit.
- Uso de Metasploit.
- Escaneos y descubrimiento de equipos con nmap.
- Enumeracion de SMB. 
- Explotacion de Vulnerabilidades comunes SMB.
- Password Spraying
- Servicios Web Comunes
- Enumeracion Y ataque a Wordpress
- Enumeracion Y ataque a Joomla
- Fuerza Bruta a SMB, SSH, RDP, Wordpress, Joomla.
- Busqueda de Exploits con Searchsploit
- FTP 
- Fuzzing a subdirectorios de una web.
- BurpSuite Basico.
- Escalada de Privilegios --> SUDO, Binarios, Tareas Cron, Version de Kernel, Capabilities

>Importante: Recomiendo aprender a usar burpsuite, por si tienes que explotar alguna Vulnerabilidad web de forma manual, ya que a mi no me funciono bien el metasploit para esto.

Bueno, esto es todo lo que yo use para el examen, ten en cuenta que te puede tocar un entorno difierente y diferentes servicios, asi que lo recomendable es que te hagas el curso de INE, si no tienes una buena base.

Gracias Por leer.

