Configurar logging en modo DEBUG en BIND (named)
===============================================

Comandos para activar el log::

   rndc querylog    # Alterna entre ON/OFF
   rndc status | grep query

1. Configurar logging en modo DEBUG
----------------------------------

Agrega un nuevo canal con nivel ``severity debug 3`` (o mayor para más detalle) en ``/etc/named.conf``:

.. code-block:: bash

   sudo nano /etc/named.conf

.. code-block:: conf

   logging {
       // Canal para consultas DNS (query logging)
       channel query_log {
           file "/var/named/data/query.log" versions 3 size 20m;
           severity dynamic;
           print-time yes;
       };
       category queries { query_log; };

       // Canal para logs DEBUG
       channel debug_log {
           file "/var/named/data/debug.log" versions 3 size 50m;
           severity debug 3;  // Nivel de debug (1-99)
           print-time yes;
           print-severity yes;
           print-category yes;
       };
       category default { debug_log; };
       category general { debug_log; };
       category resolver { debug_log; };
       category security { debug_log; };
   };

Categorías útiles para DEBUG:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

+-------------------+-----------------------------------------+
| Categoría         | Descripción                             |
+===================+=========================================+
| ``default``       | Logs generales de BIND                  |
+-------------------+-----------------------------------------+
| ``general``       | Inicio/configuración                    |
+-------------------+-----------------------------------------+
| ``resolver``      | Consultas recursivas                    |
+-------------------+-----------------------------------------+
| ``security``      | Alertas de seguridad                    |
+-------------------+-----------------------------------------+
| ``dnssec``        | Validación DNSSEC                       |
+-------------------+-----------------------------------------+
| ``update``        | Actualizaciones dinámicas               |
+-------------------+-----------------------------------------+
| ``xfer-in/out``   | Transferencias de zona                  |
+-------------------+-----------------------------------------+

2. Reiniciar BIND
-----------------

.. code-block:: bash

   sudo systemctl restart named

3. Verificar logs en tiempo real
-------------------------------

.. code-block:: bash

   sudo tail -f /var/named/data/debug.log

Ejemplo de salida DEBUG:

.. code-block:: text

   15-Jul-2024 14:30:45.123 client @0x7f8e3c0008f0 192.168.1.10#54321 (example.com): query: example.com IN A +E(0)K (192.168.1.10)
   15-Jul-2024 14:30:45.124 resolver: fetch completed for 'example.com/A' at 192.168.1.1#53

4. Ajustar nivel de debug dinámicamente
--------------------------------------

.. code-block:: bash

   sudo rndc trace 3   # Activa debug nivel 3
   sudo rndc notrace   # Desactiva debug

Niveles de debug:
~~~~~~~~~~~~~~~~~

- ``1-3``: Bajo (eventos importantes)
- ``5-10``: Medio (consultas + respuestas)
- ``20+``: Alto (detalles internos, muy verboso)

5. Rotación de logs (logrotate)
-------------------------------

.. code-block:: bash

   sudo nano /etc/logrotate.d/named-debug

.. code-block:: conf

   /var/named/data/debug.log {
       daily
       rotate 7
       compress
       delaycompress
       missingok
       notifempty
       postrotate
           /usr/bin/systemctl reload named > /dev/null 2>&1 || true
       endscript
   }

Aplicar cambios:

.. code-block:: bash

   sudo logrotate -f /etc/logrotate.d/named-debug
