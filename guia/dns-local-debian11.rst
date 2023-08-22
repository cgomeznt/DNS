Instalar DNS bind9 Debian 11
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

Editar el repositorio.::

	# vi /etc/apt/sources.list
	deb http://deb.debian.org/debian bullseye main contrib non-free
	deb http://security.debian.org/debian-security bullseye-security main
	deb-src http://security.debian.org/debian-security bullseye-security main
	deb http://backports.debian.org/debian-backports bullseye main

Actualizamos::

	# apt-get update
	
Instalamos los paquetes::

	apt-get install bind9 bind9utils bind9-doc dnsutils


Configurar un el forwarders 
------------------------------
Editar el archivo named.conf.options.::

	vi /etc/bind/named.conf.options
	#agregar estas lineas
	forwarders {
		#DNS de cantv.net
		200.44.32.12;
		200.11.248.12;
	};

Reiniciamos::

	systemctl restart bind9

Aseguramos que en el archivo host no tengamos otro DNS::

	# vi /etc/resolv.conf

	domain localdomain
	search localdomain
	nameserver 192.168.0.5

	
Para activar más detalle en los LOGs:

	# rndc querylog
	# tail -f /var/log/syslog

Si lo queremos apagar 

	# rndc querylog


hacemos pruebas con el ping o de dig

Vemos los includes que exista en el archivo::

	# cat /etc/bind/named.conf
	// This is the primary configuration file for the BIND DNS server named.
	//
	// Please read /usr/share/doc/bind9/README.Debian.gz for information on the
	// structure of BIND configuration files in Debian, *BEFORE* you customize
	// this configuration file.
	//
	// If you are just adding zones, please do that in /etc/bind/named.conf.local

	include "/etc/bind/named.conf.options";
	include "/etc/bind/named.conf.local";
	include "/etc/bind/named.conf.default-zones";


Configurar el archivo de Zona DNS Local, crear las zonas en el archivo named.conf.local.::

	# vi /etc/bind/named.conf.local
	//
	// Do any local configuration here
	//

	// Consider adding the 1918 zones here, if they are not used in your
	// organization
	//include "/etc/bind/zones.rfc1918";

	zone "e-deus.online" {
			type master;
			file "/etc/bind/db.e-deus.online";
	};

	zone "1.168.192.in-addr.arpa" {
		type master;
		file "/etc/bind/db.0.168.192";
	};

Verificamos el funcionamiento con::
	
	# named-checkconf

Ahora creamos el archivo de Zona, con los registros necesarios::

	# vi /etc/bind/db.e-deus.online
	;
	; BIND zone file for ns.e-deus.online
	;

	$TTL    3D
	@       IN      SOA     ns.e-deus.online.    root.e-deus.online. (
								2023111104      ; serial
								8H              ; refresh
								2H              ; retry
								4W              ; expire
								1D )            ; minimum
	;
						NS      ns              ; Inet address of name server
						MX      10 mail         ; Primary mail exchanger
	@               A       192.168.0.5
	ns              A       192.168.0.5
	mail            A       192.168.0.5
	www             A       192.168.0.5
	server          A       192.168.0.5
	proxy           A       192.168.0.101
	router          A       192.168.1.1     ; router ADSL
	gateway         CNAME   router


Reiniciamos::

	systemctl restart bind9

Verificamos el archivo de configuración::

	# named-checkzone e-deus.online /etc/bind/db.e-deus.online
	zone e-deus.online/IN: loaded serial 2023111104
	OK

	
Realizamod pruebas con el dig::

	# dig @192.168.0.5 e-deus.online SOA +noall +answer
	e-deus.online.          259200  IN      SOA     ns.e-deus.online. root.e-deus.online. 2023111104 28800 7200 2419200 86400

	# dig @192.168.0.5 e-deus.online NS +noall +answer
	e-deus.online.          259200  IN      NS      ns.e-deus.online.

	# dig @192.168.0.5 e-deus.online +noall +answer
	e-deus.online.          259200  IN      A       192.168.0.5

	# dig @192.168.0.5 server.e-deus.online +noall +answer
	server.e-deus.online.   259200  IN      A       192.168.0.5

	# dig @192.168.0.5 router.e-deus.online +noall +answer
	router.e-deus.online.   259200  IN      A       192.168.1.1

	# dig @192.168.0.5 gateway.e-deus.online +noall +answer
	gateway.e-deus.online.  259200  IN      CNAME   router.e-deus.online.
	router.e-deus.online.   259200  IN      A       192.168.1.1

