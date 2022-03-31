Ver y limpiar el Cache de Bind9 DNS
=====================================

El cache se puede exportar a un archivo dump::

	# ls -l /var/named/data/
	total 40
	-rw-r--r-- 1 named named 40036 Mar 30 20:36 named.run


View cache::

	# rndc dumpdb -cache

Genera el archivo /var/cache/bind/named_dump.db. ::

	# grep gnu.org /var/named/data/cache_dump.db
	gnu.org.                86358   NS      ns1.gnu.org.
		                86358   NS      ns2.gnu.org.
		                86358   NS      ns3.gnu.org.
	ns1.gnu.org.            86358   A       208.118.235.164
	ns2.gnu.org.            86358   A       87.98.253.102
	ns3.gnu.org.            86358   A       46.43.37.70

Clear cache::

	# rndc flush


Luego reload bind::

	# rndc reload
	server reload successful

Luego de limpiar el cache y se ejecutamos el dump, veremos que ya no estan los answer::

	# rndc dumpdb -cache

	# cat /var/named/data/cache_dump.db   
	;
	; Start view _default
	;
	;
	; Cache dump of view '_default' (cache _default)
	;
	$DATE 20160824004622
	;
	; Address database dump
	;
	;
	; Unassociated entries
	;
	;
	; Bad cache
	;
	;
	; Start view _bind
	;
	;
	; Cache dump of view '_bind' (cache _bind)
	;
	$DATE 20160824004622
	;
	; Address database dump
	;
	;
	; Unassociated entries
	;
	;
	; Bad cache
	;
	; Dump complete
