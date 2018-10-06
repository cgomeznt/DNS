Activar los LOGs para BIND9
=============================

Activar los LOGS
++++++++++++++++

Tipee el siguiente comando como root para activar los log y observar los query DNS.::

	# rndc querylog

Visualizar los LOGS
+++++++++++++++++++

Para visualizar los log de los query en  /var/log/messages tipee.::

	# tail -f /var/log/messages

Apagar los LOGS
+++++++++++++++++

Tipee el siguiente comando como root para apagar los log del DNS.::
	
	# rndc querylog

