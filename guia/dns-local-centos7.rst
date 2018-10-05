Configurar BIND en una Red Privada en Centos 7
++++++++++++++++++++++++++++++++++++++++++++++++

Una parte importante de la administración de la configuración del servidor y la infraestructura incluye mantener una forma fácil de buscar interfaces de red y direcciones IP por nombre, mediante la configuración de un Sistema de nombres de dominio (DNS) adecuado. El uso de nombres de dominio completos (FQDN), en lugar de direcciones IP, para especificar direcciones de red facilita la configuración de servicios y aplicaciones, y aumenta la capacidad de mantenimiento de los archivos de configuración. La configuración de su propio DNS para su red privada es una excelente manera de mejorar la administración de sus servidores.

Recuerde el firewall y el selinux

export LANG=en_US.utf8 LC_ALL=en_US.utf8


Instalar BIND en DNS Servers
+++++++++++++++++++++++++++++++

Realizamos en todos los servidores que estaran en la artilleria de DNS.::

	# yum install -y bind bind-utils


Configurar Primary DNS Server
+++++++++++++++++++++++++++++

La configuración de BIND consiste en varios archivos, que se incluyen desde el archivo de configuración principal, named.conf. Estos nombres de archivo comienzan con "named" porque ese es el nombre del proceso que BIND ejecuta. Comenzaremos con la configuración del archivo de opciones.

Realizamos siempre un respaldo de los archivos de configuracón.::

	# cp -p /etc/named.conf /etc/named.conf.orig

Editamos el archivo named.conf.::

	# vi /etc/named.conf

Sobre el bloque de opciones existente, cree un nuevo bloque de ACL llamado "trusted"". Aquí es donde definiremos la lista de clientes a los que permitiremos consultas de DNS recursivas (es decir, sus servidores que están en el mismo centro de datos que nuestro DNS).::

	acl "trusted" {
		10.128.10.11;    # ns1 - can be set to localhost
		10.128.20.12;    # ns2
		10.128.100.101;  # host1
		10.128.200.102;  # host2
	};


Ahora que tenemos nuestra lista de clientes DNS confiables, editar el bloque de opciones. Agregue la dirección IP privada de ns1 a la directiva de puerto de escucha 53, y comente la línea de escucha en v6.::

	options {
		listen-on port 53 { 127.0.0.1; 192.168.1.210; };
		# listen-on-v6 port 53 { ::1; };
	[...]


Cambie la directiva de transferencia permitida de "none" a la dirección IP privada del ns2. Además, cambie la directiva allow-query de "localhost" a "trusted"::

		allow-transfer { 192.168.0.21; };      	# disable zone transfers by default
		allow-query { trusted; };  		# allows queries from "trusted" clients
	[...]

Al final del archivo agregamos esta linea.::

	include "/etc/named/named.conf.local";

Asi quedaria el archivo /etc/named.conf en el Primary Server DNS.::

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

	acl "trusted" {
		192.168.1.210;    # ns1 - can be set to localhost
		192.168.0.21;    # ns2
		192.168.1.66;  # host1
	};

	options {
		listen-on port 53 { 127.0.0.1; 192.168.1.210; };
		# listen-on-v6 port 53 { ::1; };
		directory 	"/var/named";
		dump-file 	"/var/named/data/cache_dump.db";
		statistics-file "/var/named/data/named_stats.txt";
		memstatistics-file "/var/named/data/named_mem_stats.txt";
		#allow-query     { localhost; };

		allow-transfer { 192.168.0.21; };      # disable zone transfers by default
		allow-query { trusted; };  # allows queries from "trusted" clients

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

		dnssec-enable yes;
		dnssec-validation yes;

		/* Path to ISC DLV key */
		bindkeys-file "/etc/named.iscdlv.key";

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
	include "/etc/named/named.conf.local";



Configurar el Local File
++++++++++++++++++++++++

Ahora en el server ns1 vamos a configurar el archivo /etc/named/named.conf.local, este archivo tendra donde estaran los archivos de la zona y zonas reversas, y como se debe comportar el DNS, es decir si es Master o Esclavo para estas zonas.::

	# vi /etc/named/named.conf.local

	zone "rapidpago.local" {
	    type master;
	    file "/etc/named/zones/db.rapidpago.local"; # zone file path
	};

	zone "168.192.in-addr.arpa" {
	    type master;
	    file "/etc/named/zones/db.168.192";  # 192.168.1.0/24 subnet
	};


Creamos la Forward Zone File
++++++++++++++++++++++++++++


El archivo de zona de reenvío es donde definimos los registros de DNS para las búsquedas de DNS hacia adelante. Es decir, cuando el DNS recibe una consulta de nombre, "host1.rapidpago.local" por ejemplo, buscará en el archivo de la zona hacia adelante para resolver la dirección IP privada correspondiente del host1.::

	# mkdir  /etc/named/zones


	# vi /etc/named/zones/db.rapidpago.local

	@       IN      SOA     ns1.rapidpago.local. admin.rapidpago.local. (
		      21         ; Serial
		     604800     ; Refresh
		      86400     ; Retry
		    2419200     ; Expire
		     604800 )   ; Negative Cache TTL

	; name servers - NS records
	    IN      NS      ns1.rapidpago.local.
	    IN      NS      ns2.rapidpago.local.

	; name servers - A records
	ns1.rapidpago.local.          IN      A       192.168.1.210
	ns2.rapidpago.local.          IN      A       192.168.0.21

	; 192.168.1.0/24 192.168.1.0/24 - A records
	ldapsrv1.rapidpago.local.          IN      A       192.168.1.210
	srvscmutils.rapidpago.local.          IN      A       192.168.0.21
	scmdebian.rapidpago.local.          IN      A       192.168.1.66
	srvscm02.rapidpago.local.        IN      A      192.168.1.54
	srvscm03.rapidpago.local.        IN      A      192.168.1.11
	srvscm04.rapidpago.local.        IN      A      192.168.0.4



Crear la  Reverse Zone File(s)
++++++++++++++++++++++++++++++

El archivo de zona inversa es donde definimos registros PTR de DNS para búsquedas DNS inversas. Es decir, cuando el DNS recibe una consulta por la dirección IP, "192.168.1.66" por ejemplo, buscará en el (los) archivo(s) de zona inversa para resolver el FQDN correspondiente, "host1.rapidpago.local" en este caso .

En ns1, para cada zona inversa especificada en el archivo named.conf.local, cree un archivo de zona inversa.

Edite el archivo de zona inversa que corresponde a la(s) zona(s) inversa(s) definidas en named.conf.local.::

	# vi /etc/named/zones/db.168.192


	@       IN      SOA     ns1.rapidpago.local. admin.rapidpago.local. ( 
		                      3         ; Serial
		                 604800         ; Refresh
		                  86400         ; Retry
		                2419200         ; Expire
		                 604800 )       ; Negative Cache TTL

	; name servers - NS records
	      IN      NS      ns1.rapidpago.local.
	      IN      NS      ns2.rapidpago.local.

	; PTR Records
	210.1   IN      PTR     ns1.rapidpago.local.    ; 192.168.1.210
	21.0   IN      PTR     ns2.rapidpago.local.    ; 192.168.0.21
	210.1   IN      PTR     ldapsrv1.rapidpago.local. ; 192.168.1.210
	21.0    IN      PTR     srvscmutils.rapidpago.local. ; 192.168.0.21
	54.1    IN      PTR     srvscm02.rapidpago.local. ; 192.168.1.54
	11.1    IN      PTR     srvscm03.rapidpago.local. ; 192.168.1.11
	4.0     IN      PTR     srvscm04.rapidpago.local. ; 192.168.0.4


Chequeamos de BIND Configuration Syntax
++++++++++++++++++++++++++++++++++++++++


Ejecute el siguiente comando para verificar la sintaxis de los archivos named.conf, si todo esta bien no muestra nada.::

	# named-checkconf


El comando named-checkzone se puede usar para verificar la corrección de sus archivos de zona. Su primer argumento especifica un nombre de zona y el segundo argumento especifica el archivo de zona correspondiente, ambos definidos en named.conf.local.::

	# named-checkzone rapidpago.local /etc/named/zones/db.rapidpago.local 
	/etc/named/zones/db.rapidpago.local:1: no TTL specified; using SOA MINTTL instead
	zone rapidpago.local/IN: loaded serial 21
	OK

Verificamos tambien la zona inversa.::

	# named-checkzone 168.192.in-addr.arpa /etc/named/zones/db.168.192 
	/etc/named/zones/db.168.192:1: no TTL specified; using SOA MINTTL instead
	zone 168.192.in-addr.arpa/IN: loaded serial 21
	OK


Start BIND
+++++++++++

Iniciamos el BIND.::

	# systemctl start named

	# systemctl enable named

	# systemctl status named -l


Configurar Secondary DNS Server
++++++++++++++++++++++++++++++

Realizamos siempre un respaldo de los archivos de configuracón.::

	# cp -p /etc/named.conf /etc/named.conf.orig

Editamos el archivo named.conf.::

# vi /etc/named.conf

Al igual que el Primary Server DNS. Aquí es donde definiremos la lista de clientes a los que permitiremos consultas de DNS recursivas.::

	acl "trusted" {
		192.168.1.210;    # ns1 - can be set to localhost
		192.168.0.21;    # ns2
		192.168.1.66;  # host1
	};

Ahora que tenemos nuestra lista de clientes DNS confiables, editar el bloque de opciones. Agregue la dirección IP privada de ns1 a la directiva de puerto de escucha 53, y comente la línea de escucha en v6.::

	options {
		listen-on port 53 { 127.0.0.1; 192.168.0.21; };
		# listen-on-v6 port 53 { ::1; };
	[...]


Aqui sino vamos a permitir Transferencia de Zona como lo hacmeos en el Primary server DNS, cambie la directiva allow-query de "localhost" a "trusted".::

        allow-query { trusted; };  		# allows queries from "trusted" clients
	[...]

Al final del archivo colocamos.::

	include "/etc/named/named.conf.local";

Asi quedaria el archivo /etc/named.conf en el Primary Server DNS.::

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

	acl "trusted" {
		192.168.1.210;    # ns1 - can be set to localhost
		192.168.0.21;    # ns2
		192.168.1.66;  # host1
	};


	options {
		listen-on port 53 { 127.0.0.1; 192.168.0.21; };
		# listen-on-v6 port 53 { ::1; };
		directory 	"/var/named";
		dump-file 	"/var/named/data/cache_dump.db";
		statistics-file "/var/named/data/named_stats.txt";
		memstatistics-file "/var/named/data/named_mem_stats.txt";
		# allow-query     { localhost; };
		allow-query { trusted; };  		# allows queries from "trusted" clients	

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

		dnssec-enable yes;
		dnssec-validation yes;

		/* Path to ISC DLV key */
		bindkeys-file "/etc/named.iscdlv.key";

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
	include "/etc/named/named.conf.local";


Configurar el Local File
++++++++++++++++++++++++

Ahora en el server ns2 vamos a configurar el archivo /etc/named/named.conf.local, este archivo tendra donde estaran los archivos de la zona y zonas reversas, y como se debe comportar el DNS, es decir si es Master o Esclavo para estas zonas.::

	# vi /etc/named/named.conf.local

	zone "rapidpago.local" {
	    type slave;
	    file "/etc/named/zones/db.rapidpago.local"; # zone file path
	    masters { 192.168.1.210; };  # ns1 private IP
	};

	zone "168.192.in-addr.arpa" {
	    type slave;
	    file "/etc/named/zones/db.168.192";  # 192.168.1.0/24 subnet
	    masters { 192.168.1.210; };  # ns1 private IP
	};



Chequeamos de BIND Configuration Syntax
++++++++++++++++++++++++++++++++++++++++


Ejecute el siguiente comando para verificar la sintaxis de los archivos named.conf, si todo esta bien no muestra nada.::

	# named-checkconf


El comando named-checkzone en ns2 no aplica porque los archivos de configuracón se encuentran en el Primary Server DNS, ns1.::


Start BIND
+++++++++++

Iniciamos el BIND.::

	# systemctl start named

	# systemctl enable named

	# systemctl status named -l


Verificar en los Clientes
+++++++++++++++++++++++++


Antes de que todos sus servidores en la ACL "trusted" puedan consultar sus servidores DNS, debe configurar cada uno de ellos para usar ns1 y ns2 como servidores de nombres. Este proceso varía según el sistema operativo, pero para la mayoría de las distribuciones de Linux  implica agregar sus servidores de nombres al archivo /etc/resolv.conf.::

	# vi /etc/resolv.conf

	search rapidpago.local
	nameserver 192.168.1.210
	nameserver 192.168.0.21

Tambien podria agregar en 
/etc/sysconfig/network-scripts/ifcfg-enp0s#, una(s) lineas con DNS# y la direccion IP de los Server DNS.::

	[...]
	DNS1=192.168.1.210
	DNS2=192.168.0.21
	[...]



Test en los Clientes
++++++++++++++++++++++++

Hay dos grandes comandos que son **dig** y **nslookup** que nos ayuda a verifiace el funcionamiento del DNS en los clientes y por supuesto finalizamos con ping.::

Verificamos con nslookup la zona directa.::

	# nslookup scmdebian
	Server:		192.168.0.21
	Address:	192.168.0.21#53

	Name:	scmdebian.rapidpago.local
	Address: 192.168.1.66


Verificamos con nslookup la zona reversa.::

	# nslookup 192.168.1.66
	Server:		192.168.0.21
	Address:	192.168.0.21#53

	66.1.168.192.in-addr.arpa	name = scmdebian.rapidpago.local.

Verificamos con dig la zona directa.::

	# dig scmdebian.rapidpago.local

	; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7_5.1 <<>> scmdebian.rapidpago.local
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51844
	;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

	;; OPT PSEUDOSECTION:
	; EDNS: version: 0, flags:; udp: 4096
	;; QUESTION SECTION:
	;scmdebian.rapidpago.local.	IN	A

	;; ANSWER SECTION:
	scmdebian.rapidpago.local. 604800 IN	A	192.168.1.66

	;; AUTHORITY SECTION:
	rapidpago.local.	604800	IN	NS	ns2.rapidpago.local.
	rapidpago.local.	604800	IN	NS	ns1.rapidpago.local.

	;; ADDITIONAL SECTION:
	ns1.rapidpago.local.	604800	IN	A	192.168.1.210
	ns2.rapidpago.local.	604800	IN	A	192.168.0.21

	;; Query time: 1 msec
	;; SERVER: 192.168.0.21#53(192.168.0.21)
	;; WHEN: vie oct 05 11:06:55 -04 2018
	;; MSG SIZE  rcvd: 138

Verificamos con dig la zona reversa.::

	# dig -x 192.168.1.66

	; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7_5.1 <<>> -x 192.168.1.66
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 2805
	;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

	;; OPT PSEUDOSECTION:
	; EDNS: version: 0, flags:; udp: 4096
	;; QUESTION SECTION:
	;66.1.168.192.in-addr.arpa.	IN	PTR

	;; ANSWER SECTION:
	66.1.168.192.in-addr.arpa. 604800 IN	PTR	scmdebian.rapidpago.local.

	;; AUTHORITY SECTION:
	168.192.in-addr.arpa.	604800	IN	NS	ns1.rapidpago.local.
	168.192.in-addr.arpa.	604800	IN	NS	ns2.rapidpago.local.

	;; ADDITIONAL SECTION:
	ns1.rapidpago.local.	604800	IN	A	192.168.1.210
	ns2.rapidpago.local.	604800	IN	A	192.168.0.21

	;; Query time: 1 msec
	;; SERVER: 192.168.0.21#53(192.168.0.21)
	;; WHEN: vie oct 05 11:07:30 -04 2018
	;; MSG SIZE  rcvd: 161


Culminamos con ping para verificar, claro si no responde no significa qeu DNS este mal, solo es para confirmar que el equipo esta en linea.::

	# ping -c4 scmdebian
	PING scmdebian.rapidpago.local (192.168.1.66) 56(84) bytes of data.
	64 bytes from scmdebian.rapidpago.local (192.168.1.66): icmp_seq=1 ttl=64 time=0.157 ms
	64 bytes from scmdebian.rapidpago.local (192.168.1.66): icmp_seq=2 ttl=64 time=0.115 ms
	64 bytes from scmdebian.rapidpago.local (192.168.1.66): icmp_seq=3 ttl=64 time=0.186 ms
	64 bytes from scmdebian.rapidpago.local (192.168.1.66): icmp_seq=4 ttl=64 time=0.138 ms

	--- scmdebian.rapidpago.local ping statistics ---
	4 packets transmitted, 4 received, 0% packet loss, time 3002ms
	rtt min/avg/max/mdev = 0.115/0.149/0.186/0.026 ms

Mantenimiento del DNS
++++++++++++++++++++++++++

Mantenimiento del DNS cuando agreguen un nuevo host en la red o se elimine un host. Siempre que agregue un host a su entorno (en el mismo centro de datos), querrá agregarlo a DNS o si lo elimina. Aquí hay una lista de pasos que debe seguir:


	* Archivo de zona de reenvío: agregue un registro "A" para el nuevo host, aumente el valor de "Serie"
	* Archivo de zona inversa: agregue un registro "PTR" para el nuevo host, aumente el valor de "Serie"
	* Agregue la dirección IP privada de su nuevo host a la ACL "confiable" (named.conf.options)

Luego debemos hacer el reload.::

	# systemctl reload named

