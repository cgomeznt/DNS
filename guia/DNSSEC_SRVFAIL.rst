DNSSEC resolution on BIND in 'forwarder' mode fails with SERVFAIL or 'broken trust chain' errors
===============================

Para evitar el SERVFAIL
https://access.redhat.com/solutions/5633621

Desactivar el DNSSEC resolution restores main DNS functionality::


	dnssec-enable no;
	dnssec-validation no;
