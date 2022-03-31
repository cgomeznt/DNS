DNS Cache-Forward-local-Dinamico, Squid, Sarg, DHCP
====================================

En este laboratorio se configura:

*DNS de cache

*DNS zona local

*Squid, programa que implementa un servidor proxy para web con caché

*DansGuardian es un software de filtro de contenido

*Sarg (Squid Analysis Report Generator) es una herramienta que te permite ver donde los usuarios están conectados en Internet

*Configuracion del DHCP

*Configuracion del Dynamic DNS & DHCP 

Instalar Debian Squeeze 6.0 que tenga dos adaptadores de red.que /var y /tmp estén en particiones distintas.que la hora y fecha estén bien.

configurar la red::

	vi /etc/network/interfaces
	allow-hotplug eth0
	auto eth0
	iface eth0 inet dhcp

	allow-hotplug eth1
	auto eth1 
	iface eth1 inet static
	address 192.168.0.1
	netmask 255.255.255.0

verificar fecha y hora. Supongamos queremos poner: 27-Mayo-2007 y la hora 17:27, Esto lo haremos como root::

	# date --set "2007-05-27 17:27"
	Sun May 27 17:27:00 CET 2007

Ahora realizaremos el mismo cambio para actualizar la fecha en la BIOS::

	# hwclock --set --date="2007-05-27 17:27"

Para comprobarlo tecleamos::

	# hwclock
	Fri Feb 25 16:25:06 2000  -0.010586 seconds



Editar el repositorio::

	vi /etc/apt/sources.list
	deb http://ftp.debian.org/debian squeeze main contrib non-free
	#repositorio para sarg
	deb http://backports.debian.org/debian-backports squeeze-backports main

Actualizamos los índices de los repositorios e instalamos los paquetes::

	apt-get update
	apt-get install bind9 squid3 dansguardian apache2 sarg

Configuración y activar el forwarding entre las tarjetas::

	cat /proc/sys/net/ipv4/ip_forward
	0

Se debe cambiar al valor a 1::

	echo 1 > /proc/sys/net/ipv4/ip_forward

luego aplicamos la regla de iptables para que haga el NAT entre las tarjetas::

	iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

Luego colocamos la ip del eth1 como gateway en las estaciones de trabajo.hacemos pruebas con el ping hacia direcciones Ip como 200.44.32.12 y también ping hacia nombres de dominio como debian.org, yahoo.com como no tenemos DNS los pines con los nombres de dominio no van ha responder

Configurar un DNS de cache y forward
++++++++++++++++++++++++++++++++++

Configurar un DNS de cache y forward, Editar el archivo named.conf.options::

	vi /etc/bind/named.conf.options
	//agregar estas lineas
	forwarders {
		#DNS de cantv.net
		200.44.32.12;
		200.11.248.12;
	};

Reiniciamos el Bind9::

	/etc/init.d/bind9 restart

Hacemos pruebas con el ping hacia direcciones Ip como 200.44.32.12 y también ping hacia nombres de dominio como debian.org, yahoo.com ahora si responden los pines con los nombres de dominio.

Configurar un DNS Local, crear las zonas en el archivo named.conf.local::

	vi /etc/bind/named.conf.local
	zone "home.lan" {
	    type master;
	    file "/etc/bind/db.home.lan";
	};

	zone "1.168.192.in-addr.arpa" {
	    type master;
	    file "/etc/bind/db.1.168.192";
	};

Verificamos el funcionamiento con::

	named-checkconf

Ahora creamos la base de datos de resolución de nombres::

	vi /etc/bind/db.home.lan
	;
	; BIND zone file for home.lan
	;

	$TTL    3D
	@       IN      SOA     ns.home.lan.    root.home.lan. (
		                2010111101      ; serial
		                8H              ; refresh
		                2H              ; retry
		                4W              ; expire
		                1D )            ; minimum
	;
		        NS      ns              ; Inet address of name 
		        MX      10 mail         ; Primary mail exchanger

	ns              A       192.168.1.100
	mail            A       192.168.1.100
	home.lan.       A       192.168.1.100
	server          A       192.168.1.100
	proxy	        A       192.168.1.101
	router          A       192.168.1.1     ; router ADSL
	gateway         CNAME   router
	  
Reiniciamos Bind9::

	/etc/init.d/bind9 restart

Verificar que el fichero de configuración de la zona home.lan no contenga errores::

	named-checkzone home.lan /etc/bind/db.home.lan

Configurar en las estaciones de trabajo los DNS de la eth1 y hacer pruebas de ping hacia las entradas que creamos, como lo son; server, router, gateway, proxy recuerda tener cuidado con el sufijo de DNS debería ser "home.lan"


Squid
++++++++

Squid, programa que implementa un servidor proxy para web con caché.Configuracion del squid3. vamos al directorio::

	cd /etc/squid3
	mv squid.conf squid.conf.original
	vi squid.conf
	http_port 192.168.0.1:3128 transparent
	#http_port 3128

	cache_mem 32 MB
	cache_dir ufs /var/spool/squid3 1024 16 256

	acl manager proto cache_object
	acl localhost src 127.0.0.1/32 ::1
	acl to_localhost dst 127.0.0.0/8 0.0.0.0/32 ::1

	acl SSL_ports port 443 442 563
	acl Safe_ports port 443 442 563
	acl Safe_ports port 21
	acl Safe_ports port 3128
	acl Safe_ports port 80
	acl Safe_ports port 8080
	acl CONNECT method CONNECT
	acl my_lan src 192.168.0.0/24

	http_access allow manager localhost
	http_access deny manager
	http_access deny !Safe_ports
	http_access deny CONNECT !SSL_ports
	http_access allow localhost
	http_access allow my_lan
	http_access deny all

	hierarchy_stoplist cgi-bin ?
	coredump_dir /var/spool/squid3
	refresh_pattern ^ftp:		1440	20%	10080
	refresh_pattern ^gopher:	1440	0%	1440
	refresh_pattern -i (/cgi-bin/|\?) 0	0%	0
	refresh_pattern .		0	20%	4320
	error_directory /usr/share/squid3/errors/Spanish

Configurar en las estaciones de trabajo el proxy 192.168.0.1 con el puerte 3128, quitar la regal del iptable::

	iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

Abrir el navegador y hacer las pruebas. Recuerda que si quieres que transparent tienes que hacer esto direcciona todo que venga por el pueto 80 hacia tu server con el puerto 3128 y recuerda volver activar el gateway en los clientes::

	iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 80 -j DNAT --to 192.168.1.1:3128
	iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 3128


DansGuardian
++++++++++++++++

DansGuardian es un software de filtro de contenido. Configuración del dansguardian. vamos al directorio::

	cd /etc/dansguardian
	cp dansguardian.conf dansguardian.conf.original
	vi dansguardian.conf
	#comentar esta linea
	UNCONFIGURED - Please remove this line after configuration
	language = "spanish"
	filterport = 8080
	proxyip = 192.168.0.1
	proxyport = 3128

Reiniciamos el servicio::

	/etc/init.d/dansguardian restart

Configurar en las estaciones de trabajo el proxy 192.168.0.1 con el puerto 8080. abrir el navegador y hacer las pruebas, trata de navegar por paginas pornos


Sarg
+++++++

Sarg (Squid Analysis Report Generator) es una herramienta que te permite ver donde los usuarios están conectados en Internet. Configuracion del sarg. vamos al directorio::

	cd /etc/sarg
	cp sarg.conf sarg.conf.original
	vi sarg.conf

Solo modificamos las rutas en donde dice "squid" las debemos cambiar por "squid3"


correr sarg::

	sarg

Hacer que apache apunte hacia esta ruta para poder ver los reportes y estadísticas, /var/lib/sarg/index.html::

	vi /etc/apache2/sites-enambled/000-default

	#editar esta linea
	DocumentRoot /var/www
	#y colocarla asi
	DocumentRoot /var/lib/sarg

Reiniciar Apache::

	/etc/init.d/apache2 restart

Configurar en las estaciones de trabajo el proxy 192.168.0.1 con el puerte 8080. abrir el navegador y hacer las pruebas, ve a la ruta 192.168.0.1

Recuerda que en lugar de modificar el apache podias hacer un enlace simbolico de /var/lib/sarg/index.html en /var/www
**IMPORTANTE** sarg solo lee los reporte de squid pero hay que ejecutarlo, por lo que es recomendable hacerlo en un crontab.


DHCP
+++++++

Configuración del DHCP, vamos al directorio::

	cd /etc/default
	vi isc-dhcp-server 
	INTERFACES="eth0"

vamos al directorio::

	cd /etc/dhcp
	cp dhcpd.conf dhcpd.conf.original
	vi dhcpd.conf
	option domain-name "home.lan";
	option domain-name-servers 192.168.0.2, 192.168.0.1;
	default-lease-time 600;
	max-lease-time 7200;
	authoritative;
	subnet 192.168.0.0 netmask 255.255.255.0 {
	  range 192.168.1.101 192.168.1.250;
	  option domain-name-server 192.168.0.2, 192.168.0.1;
	  option domain-name "home.lan";
	  option routers 192.168.0.1;
	  option broadcast-address 192.168.0.255;
	}
	host desktop {
	  hardware ethernet 01:23:45:67:89:10;
	  fixed-address 192.168.1.2;
	}
	host laptop {
	  hardware ethernet 01:23:45:67:89:11;
	  fixed-address 192.168.1.3;
	}

Reiniciar el servicio dhcp::

	/etc/init.d/isc-dhcp-server restart



Configuración del Dynamic DNS & DHCP. Actualiza primero los paquetes::

	aptitude remove bind dhcp 
	aptitude install bind9 dhcp3-server 

vamos al directorio::

	cd /etc/bind
	vi named.conf
	#agregamos estas lineas
	controls {
	     inet 127.0.0.1 allow {localhost; } keys { "rndc-key"; }; 
	}; 

Este otro archivo::

	vi named.conf.local
	#agregamos esta linea, tomando en cuenta que bind9 ya lo habiamos configurado.
	include "/etc/bind/rndc.key"; 

el archivo /etc/bind/rndc.key. Contine la llave, Si el DHCP server y DNS no están en la misma maquina, se deberá copiar ese archivo en le servidor DHCP. Los db files de la zona se crean tal cual como los creamos anteriormente. vamos al directorio::

	cd /etc/default
	vi isc-dhcp-server 
	INTERFACES="eth0"

Vamos al directorio::

	cd /etc/dhcp
	vi dhcpd.conf
	# Basic stuff to name the server and switch on updating 
	server-identifier server; 
	ddns-updates on; 
	ddns-update-style interim; 
	ddns-domainname "network.athome."; 
	ddns-rev-domainname "in-addr.arpa.";
	ignore client-updates;

	 # This is the key so that DHCP can authenticate it's self to BIND9 
	include "/etc/bind/rndc.key";

	# This is the communication zone 
	zone home.lan. { 
	    primary 127.0.0.1; 
	    key rndc-key; 
	} 

	# Normal DHCP stuff 
	option domain-name "home.lan."; 
	option domain-name-servers server.home.lan
	#option domain-name-servers 192.168.0.2, 192.168.0.1; 
	option ntp-servers 192.168.0.60; 
	#option ip-forwarding off; 
	default-lease-time 600; 
	max-lease-time 7200; 
	authoritative; 
	log-facility local7; 

	subnet 192.168.0.0 netmask 255.255.255.0 {
	range 192.168.0.101 192.168.0.250; 
	option domain-name-servers 192.168.0.2, 192.168.0.1;
	option domain-name "home.lan."; 
	option routers 192.168.0.1;
	option broadcast-address 192.168.0.255; 
	#allow unknown-clients; 
	zone 0.168.192.in-addr.arpa. { 
	    primary 192.168.0.60; 
	    key "rndc-key"; 
	} 
	zone home.lan. { 
	    primary 192.168.0.60; 
	    key "rndc-key"; 
	} 
	} 

