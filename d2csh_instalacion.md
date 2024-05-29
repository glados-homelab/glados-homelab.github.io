# Configurar IP dinámica en registros DNS Cloudflare con **d2c.sh**

Recientemente me he visto en la necesidad un servicio de estas características. He realizado la portabilidad de ISP de DIGI a Movistar. Anteriormente DIGI (a pesar de trabajar con CG-NAT) me ofrecía una IPv4 estática por el módico precio de 1€ al mes. Sin embargo, Movistar me pide unos inaceptables 30€ por una dirección IP publica fija, cuando con DIGI pagaba la gran totalidad de 26€ por todo el servicio de fibra 1GB además la dirección publica.

Después de que Movistar me haya cambiado la IP 6 veces en el plazo de 2 semanas, he decidido que es hora de actuar.

## Requisitos previos

En mi caso realizo la instalación sobre un contenedor LXC en un nodo Proxmox. Este contenedor tiene instalado Ubuntu 22.04. Estos son los recursos que le asigné:
- 1 núcleo
- 256MB de memoria
- 5GB de almacenamiento
- IP estática en la red interna

También requiere las siguientes dependencias (si no ese encuentran ya instaladas):

- [yq](https://github.com/mikefarah/yq)
- [curl](https://curl.se/download.html)
- git (opcional)

El paquete de **curl** y **git** se pueden instalar mediante el gestor de paquetes APT o mediante su página web.

````bash
sudo apt update
sudo apt install curl git
````

Sin embargo **yq** debe ser instalado mediante GitHub. Podemos usar el siguiente comando indicado en el repositorio para descargar e instalar la última versión.

````bash
wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/bin/yq

chmod +x /usr/bin/yq
````

Con esto ya tenemos todos los pre requisitos para la instalación de d2c.sh

## Instalación

1. Clonamos el repositorio

````bash
git clone https://github.com/ddries/d2c.sh.git
````
Esto clonará el repositorio  nos descargará todos los archivos en el directorio ````.\d2c.sh```` (**OJO**, porque aunque el nombre acabe en ``.sh``, es un directorio)

2. Entramos en el directorio y ejecutamos el script de instalación.

````bash
cd d2c.sh
.\install
````

Al ejecutar este, copiará el sd2c.sh en ````/usr/local/bin```` y le otorgará permisos de ejecución. Nos sacará lo siguiente por pantalla:

````
Successfully installed d2c.sh into /usr/local/bin.
Please, run d2c.sh from command-line before scheduling any cronjob.
Help: \`d2c.sh --help\` or \`d2c.sh -h\` or \`d2c.sh help\`.
````

Esto quiere decir que la instalación ha sido exitosa.

## Configuración

Ahora que el script como tal ya está descargado e instalado, tenemos que configurar.

1.  Debemos de ejecutar el script por primera vez para que cree el directorio de configuración.

````bash
/bin/bash /usr/local/bin/d2c.sh
````

Nos dará feedback por pantalla:
````bash
Directory: /etc/d2c/ does not exist.
Creating...
Created /etc/d2c/. Please, fill /etc/d2c/d2c.toml.
````

El script nos crea atomaticamente su carpeta de configuración en ````/etc/d2c````, pero no nos crea el archivo ````d2c.toml````. Lo crearemos y lo editaremos con nuestro editor de preferencia, en mi caso vim. En caso de que seas principiante en el uso de la terminal linux, te recomiendo usar nano.

````bash
touch /etc/d2c/d2c.toml
vim /etc/d2c/d2c.toml
````

Aquí pegaremos la plantilla que nos deja el autor en el repositorio.

````toml
[api]
zone-id = "aaa" # your dns zone id
api-key = "bbb" # your api key with dns records permissions

[[dns]]
name = "dns1.example.com" # dns name
proxy = true # proxied by cloudflare?

[[dns]]
name = "dns2.example.com"
proxy = false
````

Vamos a explicar esta configuración

### Sintaxis del fichero ````d2c.toml````

````toml
[api]
zone-id = "aaa" # your dns zone id
api-key = "bbb" # your api key with dns records permissions
````
- **zone-id** --> Id de la zona DNS a administrar. Lo puedes encontrar siguiendo lo siguientes pasos:
    1. Abre el panel de Cloudflare y selecciona la zona que quieres administrar
    
    !["image01"](\\images\d2csh_instalacion\image01.png)


    2. En el panel de la izquierda nos cercioramos que estamos en la pestaña "Overview" y bajamos. En el panel lateral derecho observamos la sección **"API"**. En ella tenemos el campo "Zone ID", el cual deberemos de copiar e introducir en el campo "zone-id" del fichero de configuración.
    !["image02"](\\images\d2csh_instalacion\image02.png)


- **api-key** --> El token para acceder a la API. Deberemos de crearlo desde Cloudflare con los permisos adecuados:
    1. En el panel de Cloudflare iremos arriba a la derecha, abrimos el desplegable y hacemos clic en "My Profile". 
    [""](\\image\d2csh_instalacion\image03.png) 

    2. En el menú de la izquierda haremos clic en "API Tokens"
    [""](\\image\d2csh_instalacion\image04.png) 

    3. Una vez en "API Tokens", le damos al botón "Create Token"
     
     .

     .

     .

     .

     .

````toml
[[dns]]
name = "dns1.example.com" # dns name
proxy = true # proxied by cloudflare?
````

- **name** --> El nombre FQDN (Full Qualified Domain Name) que queremos editar, es decir, el nombre completo DNS, con el TLD y subdominio.

- **proxy** --> Aquí deberemos indicar si esta entrada DNS está configurada para hacer proxy a traves de los servidores de Cloudflare o apunta directamente a la IP de destino.

Repetiremos una entrada como la anterior para todos los registros que queramos actualizar, teniendo siempre en cuenta que **[[DNS]]** no debe ser alterado, ni el texto ni los **dobles corchetes**.

Cuando hayamos realizado toda la configuración guardaremos el archivo.

Ejecutamos el script para ver si funciona correctamente. 

````bash
root@dyndns:~# /usr/local/bin/d2c.sh
[d2c.sh] OK: dns1.example.com
[d2c.sh] OK: dns2.example.com
````
Si el script nos retorna la salida de arriba, los datos deberían de haberse actualizado correctamente en Cloudflare.

````bash
root@dyndns:~# /usr/local/bin/d2c.sh
[d2c.sh] dns1.example.com did not change
[d2c.sh] dns2.example.com did not change
````

Si los nombres DNS ya estaban apuntando a nuestra dirección actual, recibiremos la anterior salida. Esto quiere decir que aunque no los ha actualizado (porque ya eran correctos), la configuración es correcta y conecta con Cloudflare.

## Configuración crontab
Ahora que ya tenemos todo funcionando, debemos de hacer que se ejecute con cierta periodicidad. Para esto vamos a usar el **crontab**. Si no sabeis que es crontab, podeis aprenderlo [aquí](https://es.wikipedia.org/wiki/Cron_(Unix)).

Para editar el crontab, ejecutaremos:
````bash
crontab -e 
````

Esto nos permitirá editar el archivo crontab. Al final del archivo añadiremos la siguiente linea, donde x = cada cuantos minutos debe ejecutarse:

````bash
*/x * * * * /usr/local/bin/d2c.sh
````

En mi caso, quiero que se ejecute cada 5 minutos. asi que escribiremos:
````bash
*/5 * * * * /usr/local/bin/d2c.sh
````
Salimos del archivo guardando los cambios.