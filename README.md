# tcp-mitm-proxy
TCP proxy con multiplexación (select), detección de destino vía SNI/SO_ORIGINAL_DST/CLI, e interceptación TLS mediante certificados propios.
# tcp-mitm-proxy

Un proxy TCP hecho a mano que intercepta tráfico HTTP y HTTPS, te deja ver cada paquete en hexadecimal o texto, y te da la opción de editarlo, dejarlo pasar o descartarlo antes de que llegue a destino.

No es un fork de mitmproxy ni un wrapper de otra herramienta. Es socket crudo, threading y select, escrito para entender qué pasa realmente cuando un paquete viaja de un cliente a un servidor y alguien se para en el medio.

## Por qué existe

Arrancó como un ejercicio siguiendo *Black Hat Python*, pero se quedó corto rápido. La versión del libro reenvía bytes a ciegas. Esta versión decide a dónde mandar cada conexión usando tres métodos distintos según lo que tenga disponible, sabe abrir el candado de TLS para leer HTTPS en texto plano, y convierte cada paquete interceptado en algo editable en el momento, no en un log para revisar después.

## Qué hace

**Detecta el destino de tres formas distintas**, en orden de prioridad:
1. Lee el método `CONNECT` o la cabecera `Host` del tráfico HTTP/HTTPS.
2. Si no hay nada de capa 7 (por ejemplo, tráfico redirigido con iptables en un ataque de ARP spoofing), consulta `SO_ORIGINAL_DST` al kernel de Linux para recuperar el destino original de la conexión.
3. Si ninguna de las dos funciona, cae a un destino fijo pasado por línea de comandos.

**Intercepta HTTPS de verdad, no solo lo reenvía.** Cuando detecta un `CONNECT`, en vez de pasar el handshake TLS tal cual, se para en el medio con dos conexiones TLS separadas: una hacia el cliente usando un certificado propio, y otra hacia el servidor real. El resultado es que el tráfico HTTPS se puede leer y editar como si fuera texto plano.

**Deja editar cada paquete a mano.** Cuando algo pasa por el proxy, se pausa, muestra el hexdump en consola y da a elegir: aceptarlo, editarlo como texto, editarlo en hexadecimal abriendo tu editor favorito, o descartarlo directamente.

**Maneja varias conexiones a la vez** usando `select` para multiplexar sockets en vez de bloquear un hilo por conexión, con un lock para que la salida en consola no se mezcle entre paquetes de distintos clientes.

## Cómo se usa

```bash
# generar el certificado que usa el proxy para MITM en HTTPS
python3 generate_cert.py

# levantar el proxy escuchando en 127.0.0.1:9999
python3 tcp_proxy.py -lh 127.0.0.1 -lp 9999
```

Para probarlo contra un destino fijo (sin depender de detección automática):

```bash
python3 tcp_proxy.py -lp 9999 -rh example.com -rp 443
```

Y para usarlo como proxy real desde otra herramienta:

```bash
curl -x http://127.0.0.1:9999 --cacert proxy.crt https://example.com
```

Cada paquete que pase por ahí te va a preguntar qué hacer con él antes de reenviarlo.

## Estado del proyecto

Esto es una herramienta de laboratorio, no algo para apuntar a producción ni a tráfico de terceros sin permiso. El certificado que genera es autofirmado y único para todo el tráfico interceptado — todavía no genera un certificado distinto por dominio, así que un cliente que valide certificados de forma estricta (certificate pinning) lo va a rechazar. Ese es el próximo paso: una CA propia que firme certificados al vuelo por dominio, para que la intercepción sea transparente incluso para clientes más exigentes.

## Uso responsable

Esto interfiere tráfico de red y descifra HTTPS. Usalo únicamente en entornos que sean tuyos o donde tengas autorización explícita para hacer pruebas — tu propia red, una VM de laboratorio, o un engagement de pentesting con permiso por escrito. Interceptar tráfico ajeno sin autorización es ilegal en la gran mayoría de las jurisdicciones.
