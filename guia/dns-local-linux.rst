Instalar Debian Squeeze 6.0
=============================


que tenga dos adaptadores de red.
que /var y /tmp esten en particiones distintas.
que la hora y fecha esten bien.

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

verificar fecha y hora.::

	date

#Editar el repositorio.::

	vi /etc/apt/sources.list
	deb http://ftp.debian.org/debian squeeze main contrib non-free
	#repositorio para sarg
	deb http://backports.debian.org/debian-backports squeeze-backports main

	apt-get update
	apt-get install bind9 squid3 dansguardian apache2 sarg

#Configuracion y activar el forwarding entre las tarjetas.::

	cat /proc/sys/net/ipv4/ip_forward
	0

#se debe cambiar al valor a 1.::

	echo 1 > /proc/sys/net/ipv4/ip_forward

#luego aplicamos la regla de iptables para que haga el NAT entre las tarjetas.::

	iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

#Luego colocamos la ip del eth1 como gateway en las estaciones de trabajo.
#hacemos pruebas con el ping hacia direcciones Ip como 200.44.32.12
#y tambien ping hacia nombres de dominio como debian.org, yahoo.com 
#como no tenemos DNS los pines con los nombres de dominio no van ha responder

Instalamos los paqutes::

	apt-get install bind9 bind9utils bind9-doc dnsutils

#Configurar un DNS de cache
#Editar el archivo named.conf.options.::

	vi /etc/bind/named.conf.options
	#agregar estas lineas
	forwarders {
		#DNS de cantv.net
		200.44.32.12;
		200.11.248.12;
	};

	/etc/init.d/bind9 restart

#hacemos pruebas con el ping hacia direcciones Ip como 200.44.32.12
#y tambien ping hacia nombres de dominio como debian.org, yahoo.com 
#ahora si responden los pines con los nombres de dominio.

#Configurar un DNS Local
#crear las zonas en el archivo named.conf.local.::

	vi /etc/bind/named.conf.local
	zone "home.lan" {
		type master;
		file "/etc/bind/db.home.lan";
	};

	zone "1.168.192.in-addr.arpa" {
		type master;
		file "/etc/bind/db.1.168.192";
	};

#verificamos el funcionamiento con::
	
	named-checkconf

#Ahora creamos la base de datos de resolucion de nombres::

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
		            NS      ns              ; Inet address of name server
		            MX      10 mail         ; Primary mail exchanger

	ns              A       192.168.1.100
	mail            A       192.168.1.100
	home.lan.       A       192.168.1.100
	server          A       192.168.1.100
	proxy	        A       192.168.1.101
	router          A       192.168.1.1     ; router ADSL
	gateway         CNAME   router
	  

	/etc/init.d/bind9 restart

#Configurar en las estaciones de trabajo los DNS de la eth1 y hacer pruebas de ping hacia
#las entradas que creamos, como lo son; server, router, gateway, proxy
