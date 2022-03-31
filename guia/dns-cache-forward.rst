Configurar BIND Como CACHE y Forward Centos 7
++++++++++++++++++++++++++++++++++++++++++++++++

Recuerde el firewall y el selinux

export LANG=en_US.utf8 LC_ALL=en_US.utf8


Instalar BIND en DNS Servers
+++++++++++++++++++++++++++++++

Realizamos en todos los servidores que estarán en la artillería de DNS.::

	# yum install -y bind bind-utils


Configurar Primary DNS Server
+++++++++++++++++++++++++++++

La configuración de BIND consiste en varios archivos, que se incluyen desde el archivo de configuración principal, named.conf. Estos nombres de archivo comienzan con "named" porque ese es el nombre del proceso que BIND ejecuta. Comenzaremos con la configuración del archivo de opciones.

Realizamos siempre un respaldo de los archivos de configuracón.::

	# cp -p /etc/named.conf /etc/named.conf.orig

Editamos el archivo named.conf y agregamos estas lineas::

	# vi /etc/named.conf
		forwarders {
		        //DNS Google
		        8.8.8.8;
		        //DNS Freedom
		        80.80.80.80;
		        //DNS del Dominio Local
		        172.16.0.61;
		        172.16.0.3;
		};

		dnssec-enable no;
		dnssec-validation no;

Quedaría de esta forma el named.conf::


	//
	// named.conf
	//
	// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
	// server as a caching only nameserver (as a localhost DNS resolver only).
	//
	// See /usr/share/doc/bind*/sample/ for example named configuration files.
	//
	// See the BIND Administrator's Reference Manual (ARM) for details about the
	// configuration located in /usr/share/doc/bind-{version}/Bv9ARM.html

	options {
		listen-on port 53 { 127.0.0.1; };
		//listen-on-v6 port 53 { ::1; };
		directory       "/var/named";
		dump-file       "/var/named/data/cache_dump.db";
		statistics-file "/var/named/data/named_stats.txt";
		memstatistics-file "/var/named/data/named_mem_stats.txt";
		recursing-file  "/var/named/data/named.recursing";
		secroots-file   "/var/named/data/named.secroots";
		allow-query     { localhost; };
		//allow-recursion { localhost; };

		forwarders {
		        //DNS Google
		        8.8.8.8;
		        //DNS Freedom
		        80.80.80.80;
		        //DNS Credicard
		        10.132.0.61;
		        10.132.0.3;
		};
		/*
		 - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
		 - If you are building a RECURSIVE (caching) DNS server, you need to enable
		   recursion.
		 - If your recursive DNS server has a public IP address, you MUST enable access
		   control to limit queries to your legitimate users. Failing to do so will
		   cause your server to become part of large scale DNS amplification
		   attacks. Implementing BCP38 within your network would greatly
		   reduce such attack surface
		*/
		recursion yes;

		dnssec-enable no;
		dnssec-validation no;

		/* Path to ISC DLV key */
		bindkeys-file "/etc/named.root.key";

		managed-keys-directory "/var/named/dynamic";

		pid-file "/run/named/named.pid";
		session-keyfile "/run/named/session.key";
	};

	logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
	};

	zone "." IN {
		type hint;
		file "named.ca";
	};

	include "/etc/named.rfc1912.zones";
	include "/etc/named.root.key";

Verificamos el archivo de configuración::

	# named-checkconf


Activamos el log para ver todo::

	# rndc querylog

Para visualizar los log de los query en  /var/log/messages tipee::

	# tail -f /var/log/messages


Para desactivarlo lo volvemos a ejecuta, es decir, apagar los log del DNS::
	
	# rndc querylog

Iniciamos el Bind o lo recargamos si ya estaba iniciado::

	systemctl start named

	o

	systemctl reload named
