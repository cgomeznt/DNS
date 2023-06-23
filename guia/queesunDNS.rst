¿Qué es DNS? | Cómo funciona
===============================

¿Qué es DNS?
----------------

El sistema de nombres de dominio (DNS) es el directorio telefónico de Internet. Las personas acceden a la información en línea a través de nombres de dominio como nytimes.com o espn.com. Los navegadores web interactúan mediante direcciones de Protocolo de Internet (IP). El DNS traduce los nombres de dominio a direcciones IP para que los navegadores puedan cargar los recursos de Internet.

Cada dispositivo conectado a Internet tiene una dirección IP única que otros equipos pueden usar para encontrarlo. Los servidores DNS suprimen la necesidad de que los humanos memoricen direcciones IP tales como 192.168.1.1 (en IPv4) o nuevas direcciones IP alfanuméricas más complejas, tales como 2400:cb00:2048:1::c629:d7a2 (en IPv6).


¿Cómo funciona DNS?
------------------------

El proceso de solución de DNS supone convertir un nombre de servidor (como www.ejemplo.com) en una dirección IP compatible con el ordenador (como 192.168.1.1). Se da una dirección IP a cada dispositivo en Internet, y esa dirección será necesaria para encontrar el dispositivo apropiado de Internet, al igual que se usa la dirección de una calle para encontrar una casa concreta. Cuando un usuario quiere cargar una página, se debe traducir lo que el usuario escribe en su navegador web (ejemplo.com) a una dirección que el ordenador pueda entender para poder localizar la página web de ejemplo.com.

Para entender el proceso de la resolución de DNS, es importante conocer los diferentes componentes de hardware por los que debe pasar una consulta de DNS. Para el navegador web, la búsqueda de DNS se produce "en segundo plano" y no requiere ninguna interacción del ordenador del usuario, aparte de la solicitud inicial.

Hay 4 servidores DNS implicados en la carga de un sitio web:
------------------------------------------------------------

**DNS recursor**: es como un bibliotecario al que se le pide que busque un libro determinado en la biblioteca. El recursor DNS es un servidor diseñado para recibir consultas desde equipos cliente mediante aplicaciones como navegadores web. Normalmente, el recursor será el responsable de hacer solicitudes adicionales para satisfacer la consulta de DNS del cliente.

**Root nameserver**: es el primer paso para traducir (solucionar) los nombres de servidor legibles en direcciones IP. Se puede comparar a un índice en una biblioteca que apunta a diferentes estanterías de libros. Generalmente sirve como referencia de otras ubicaciones más específicas.

**TLD nameserver**: el servidor de dominio de nivel superior (TLD) se puede comparar con una estantería de libros en una biblioteca. Es el paso siguiente en la búsqueda de una dirección IP específica y aloja la última parte de un nombre de servidor (en ejemplo.com, el servidor TLD es "com").

**Authoritative nameserver**: se puede interpretar como un diccionario en una estantería de libros, en el que se puede consultar la definición de un nombre específico. El servidor de nombres autoritativo es la última parada en la consulta del servidor de nombres. Si cuenta con acceso al registro solicitado, devolverá la dirección IP del nombre del servidor solicitado al recursor de DNS (el bibliotecario) que hizo la solicitud inicial.

