# HTTP

<!--introduced_in=v0.10.0-->

> Estabilidad: 2 - Estable

Para utilizar el servidor HTTP y el cliente, uno debe requerirlo de la siguiente manera `require('http')`.

Las interfaces HTTP en Node.js están diseñadas para soportar varias características del protocolo que, tradicionalmente, han sido difíciles de utilizar. En particular, mensajes grandes y potencialmente codificados en fragmentos. La interfaz nunca almacena respuestas o peticiones enteras — el usuario puede establecer entonces un flujo continuo de datos.

Los encabezados de los mensajes HTTP se representan mediante un objeto como el siguiente:

<!-- eslint-skip -->

```js
{ 'content-length': '123',
  'content-type': 'text/plain',
  'connection': 'keep-alive',
  'host': 'mysite.com',
  'accept': '*/*' }
```

Las llaves o identificadores se escriben en minúscula. Los valores no son modificados.

Para poder soportar el espectro completo de las aplicaciones HTTP, la API HTTP de Node.js es de muy bajo nivel. Se encarga solo de manejar flujos y analizar mensajes. Puede analizar y re ordenar un mensaje en encabezado y cuerpo, pero no puede hacer lo mismo con un objeto header o un objeto body.

Consulte la sección [`message.headers`][] para conocer detalles de como los encabezados duplicados son manejados.

Los encabezados sin procesar están retenidos en la propiedad `rawHeaders`, que es un arreglo con la estructura `[key, value, key2, value2, ...]`. Por ejemplo, el objecto encabezado anterior puede tener un arreglo para la propiedad `rawHeaders` como el siguiente:

<!-- eslint-disable semi -->

```js
[ 'content-length', '123',
  'content-type', 'text/plain',
  'connection', 'keep-alive',
  'host', 'mysite.com',
  'accept', '*/*' ]
```

## Clase: http.Agent

<!-- YAML
added: v0.3.4
-->

Un `Agent` es responsable del manejo de la persistencia y reutilización de las conexiones en los clientes HTTP. Mantiene una cola de peticiones pendientes para un host definido y un puerto, reutilizando un único socket para cada una hasta que la cola se encuentra vacía. Que una petición se destruya o sea agrupada con otras, depende de la [opción](#http_new_agent_options) `keepAlive`.

Las conexiones agrupadas tienen la opcion TCP Keep-Alive habilitada, pero incluso así los servidores pueden cerrar las conexiones en espera. De ocurrir, las mismas serán removidas del grupo y una nueva conexión sera establecida cuando una nueva petición HTTP sea realizada para ese host y ese puerto específico. Los servidores también pueden denegar el permiso de permitir múltiples peticiones en una misma conexión, por lo que en este caso la conexión tendrá que ser re establecida para cada petición y no podrá ser agrupada. El `Agent` hará las peticiones a ese servidor, pero cada una será llevada a cabo en una nueva conexión.

Cuando una conexión es cerrada por el cliente o por el servidor, la misma es removida del grupo. Todos los sockets del grupo que ya no sean utilizados, serán desreferenciados para evitar que el proceso de Node.js se mantenga activo cuando no hay mas llamadas pendientes. (Consultar la sección [`socket.unref()`]).

Se considera una buena práctica destruir la instancia del `Agent` cuando ya no esta siendo utilizada, ya que los sockets que persisten consumen recursos del SO. (Consulte la sección [`destroy()`][]).

Los sockets son removidos de un agente cuando emiten un evento `'close'` o un evento `'agentRemove'`. Si la intención es mantener una petición HTTP activa por un periodo de tiempo indefinido, sin mantenerla dentro del agent se puede hacer algo como lo siguiente:

```js
http.get(options, (res) => {
  // Hacer algo
}).on('socket', (socket) => {
  socket.emit('agentRemove');
});
```

Un agent también puede ser utilizado para una petición individual. Al proveer `{agent: false}` como una opción a las funciones `http.get()` o `http.request()`, un `Agent` de uso único, con la configuración por defecto, sera utilizado para la conexión del cliente.

`agent:false`:

```js
http.get({
  hostname: 'localhost',
  port: 80,
  path: '/',
  agent: false  // crea un nuevo agente solo para esta petición
}, (res) => {
  // Hacer algo con la respuesta
});
```

### new Agent([options])

<!-- YAML
added: v0.3.4
-->

* `options` {Object} Conjunto de opciones configurables aplicables al agente. Puede contener los siguientes campos: 
  * `keepAlive` {boolean} Mantiene los sockets activos incluso cuando no hay peticiones pendientes, para que puedan ser utilizados por futuras peticiones sin tener que re-establecer una conexión TCP. **Default:**`false`.
  * `keepAliveMsecs` {number} Cuando se utiliza la opción `keepAlive`, especifica el [ delay inicial](net.html#net_socket_setkeepalive_enable_initialdelay) para los paquetes TCP Keep-Alive. Se ignora cuando la opción `keepAlive` es `false` o `undefined`. **Default:** `1000`.
  * `maxSockets` {number} Número máximo de sockets permitidos por host. **Default:** `Infinito`.
  * `maxFreeSockets` {number} Número máximo de sockets a dejar disponibles en un estado libre. Solo aplica si `keepAlive` tiene valor `true`. **Default:** `256`.

El [`http.globalAgent`][] que es utilizado por [`http.request()`][] tiene todos estos valores configurados como sus valores por defecto.

Para configurar cualquiera de ellos, se deberá crear una instancia de [`http.Agent`][].

```js
const http = require('http');
const keepAliveAgent = new http.Agent({ keepAlive: true });
options.agent = keepAliveAgent;
http.request(options, onResponseCallback);
```

### agent.createConnection(options[, callback])

<!-- YAML
added: v0.11.4
-->

* `options` {Object} Opciones que contienen los detalles de conexión. Consultar [`net.createConnection()`][] para ver el formato de las opciones
* `callback` {Function} Función callback que recibe el socket creado
* Retorna: {net.Socket}

Produce un socket/stream para ser utilizado por las peticiones HTTP.

Por defecto, esta función es la misma que [`net.createConnection()`][]. Es posible anular este método con un agente personalizado en caso de que se desee mayor flexibilidad.

Un socket/stream puede ser proporcionado de dos maneras: retornando el socket/stream desde esta función, o pasando el socket/stream como argumento a la función `callback`.

`callback` contempla el ingreso de `(err, stream)` como parámetros.

### agent.keepSocketAlive(socket)

<!-- YAML
added: v8.1.0
-->

* `socket` {net.Socket}

Se invoca cuando `socket` se desreferencia de una petición y puede ser persistido por el `Agent`. El comportamiento por defecto es:

```js
socket.setKeepAlive(true, this.keepAliveMsecs);
socket.unref();
return true;
```

Este método puede ser anulado por una subclase `Agent` particular. Si este método retorna un valor falsy, el socket sera destruido en vez de persistir para ser utilizado en la próxima petición.

### agent.reuseSocket(socket, request)

<!-- YAML
added: v8.1.0
-->

* `socket` {net.Socket}
* `request` {http.ClientRequest}

Invocado cuando `socket` se adosa a `request` luego de ser persistido por las opciones de keep-alive. El comportamiento por defecto es:

```js
socket.ref();
```

Este método puede ser anulado por una subclase particular `Agent`.

### agent.destroy()

<!-- YAML
added: v0.11.4
-->

Destruye cualquier socket que este siendo utilizado por el agent.

Generalmente, no es necesario hacer esto. De cualquier manera, si se esta utilizando un agent con `keepAlive` habilitado, entonces es mejor cerrar el agente explícitamente cuando ya no va a ser utilizado. De otra forma, los sockets pueden mantenerse habilitados por tiempo indeterminado hasta que el server los termine.

### agent.freeSockets

<!-- YAML
added: v0.11.4
-->

* {Object}

Un objeto que contiene un arreglo de sockets disponibles para ser utilizados por el agente cuando `keepAlive` se encuentra habilitado. No modificar.

### agent.getName(options)

<!-- YAML
added: v0.11.4
-->

* `options` {Object} Conjunto de opciones que contiene la información para la generación de nombres 
  * `host` {string} Un nombre de dominio o dirección IP del servidor al cual se le emitirá la petición
  * `port` {number} Puerto del servidor remoto
  * `localAddress` {string} Interfaz local a la cual se realiza en enlace cuando se emite la petición
  * `family` {integer} Debe ser 4 o 6 si su valor no es igual a `undefined`.
* Retorna: {string}

Obtiene un nombre único para un conjunto de opciones de petición, para determinar si una conexión puede ser reutilizada. Para un agente HTTP, retorna `host:port:localAddress` o `host:port:localAddress:family`. Para un agente HTTPS, el nombre incluye la Autoridad de Certificación, certificado, cifras, y otras opciones específicas a HTTPS/TLS que determinan la reusabilidad de un socket.

### agent.maxFreeSockets

<!-- YAML
added: v0.11.7
-->

* {number}

Por defecto, el valor es 256. Para agentes con `keepAlive` habilitado, define el valor máximo de sockets que quedaran abiertos en el estado libre.

### agent.maxSockets

<!-- YAML
added: v0.3.6
-->

* {number}

Por defecto, el valor es `Infinito`. Determina cuantos sockets concurrentes el agente puede tener abierto por origen. Origen es el valor de retorno de [`agent.getName()`][].

### agent.requests

<!-- YAML
added: v0.5.9
-->

* {Object}

Un objeto que contiene colas de peticiones que aún no han sido asignadas a sockets. No modificar.

### agent.sockets

<!-- YAML
added: v0.3.6
-->

* {Object}

Un objeto que contiene arreglos de sockets siendo utilizados por el agente. No modificar.

## Clase: http.ClientRequest

<!-- YAML
added: v0.1.17
-->

Este objecto es creado internamente y es el valor de retorno de [`http.request()`][]. Representa una petición *en progreso* cuyo encabezado ya se encuentra en cola. El encabezado aún es mutable utilizando las APIs de [`setHeader(name, value)`][], [`getHeader(name)`][], [`removeHeader(name)`][]. El encabezado será enviado junto con el primer fragmento de datos o cuando se invoque [`request.end()`][].

Para obtener la respuesta, se debe agregar un listener [`'response'`][] al objeto de la petición. [`'response'`][] será emitido del objecto de petición cuando los encabezados de respuesta hayan sido recibidos. El evento [`'response'`][] se ejecuta con un argumento que es una instancia de [`http.IncomingMessage`][].

Durante el evento [`'response'`][], se pueden agregar listeners al objeto de respuesta; particularmente para esperar que ocurra el evento `'data'`.

Si no se incluye código para manejar la [`'response'`][], entonces la respuesta sera descartada en su totalidad. Sin embargo, en caso de incluir código para manejar [`'response'`][], entonces los datos del objeto de respuesta **deben** ser consumidos ya sea llamando `response.read()` cuando ocurra un evento `'readable'`, o agregando una forma de manipular `'data'`, o llamando al método `.resume()`. Hasta que la data no sea consumida, el evento `'end'` no se va a disparar. También, hasta que la data no sea leída, va a consumir memoria que eventualmente puede desembocar en un error 'process out of memory'.

Node.js no comprueba si el Content-Length y la longitud del objeto body transmitido son iguales o no.

La petición implementa la interfaz [Writable Stream](stream.html#stream_class_stream_writable). Esto es un [`EventEmitter`][] con los siguientes eventos:

### Evento: 'abort'

<!-- YAML
added: v1.4.1
-->

Emitido cuando la petición ha sido cancelada por el cliente. Este evento solo es emitido en la primer llamada a `abort()`.

### Evento: 'connect'

<!-- YAML
added: v0.7.0
-->

* `response` {http.IncomingMessage}
* `socket` {net.Socket}
* `head` {Buffer}

Emitido cada vez que el servidor responde a una petición con el método `CONNECT`. Si este evento no está siendo atendido, los clientes recibiendo un método `CONNECT` cerrarán sus conexiones.

Un cliente y un servidor demostrando cómo atender el evento `'connect'`:

```js
const http = require('http');
const net = require('net');
const url = require('url');

// Crea un proxy túnel HTTP
const proxy = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('okay');
});
proxy.on('connect', (req, cltSocket, head) => {
  // conectar a un servidor de origen
  const srvUrl = url.parse(`http://${req.url}`);
  const srvSocket = net.connect(srvUrl.port, srvUrl.hostname, () => {
    cltSocket.write('HTTP/1.1 200 Connection Established\r\n' +
                    'Proxy-agent: Node.js-Proxy\r\n' +
                    '\r\n');
    srvSocket.write(head);
    srvSocket.pipe(cltSocket);
    cltSocket.pipe(srvSocket);
  });
});

// ahora el proxy está corriendo
proxy.listen(1337, '127.0.0.1', () => {

  // hacer una petición al túnel proxy
  const options = {
    port: 1337,
    hostname: '127.0.0.1',
    method: 'CONNECT',
    path: 'www.google.com:80'
  };

  const req = http.request(options);
  req.end();

  req.on('connect', (res, socket, head) => {
    console.log('conectado!');

    // hacer una petición a través de un túnel HTTP
    socket.write('GET / HTTP/1.1\r\n' +
                 'Host: www.google.com:80\r\n' +
                 'Connection: close\r\n' +
                 '\r\n');
    socket.on('data', (chunk) => {
      console.log(chunk.toString());
    });
    socket.on('end', () => {
      proxy.close();
    });
  });
});
```

### Evento: 'continue'

<!-- YAML
added: v0.3.2
-->

Emitido cuando el servidor envía una respuesta HTTP '100 Continue', normalmente porque la petición contenía 'Expect: 100-continue'. Esta es una instrucción en que el cliente debería enviar el objeto body de la petición.

### Evento: 'information'

<!-- YAML
added: v10.0.0
-->

Emitido cuando el servidor envía una respuesta 1xx (excluyendo 101 Upgrade). Este evento es emitido con un callback conteniendo un objeto con un código de estado HTTP.

```js
const http = require('http');

const options = {
  hostname: '127.0.0.1',
  port: 8080,
  path: '/longitud_peticion'
};

// Hace una solicitud
const req = http.request(options);
req.end();

req.on('information', (res) => {
  console.log(`Información recibida antes de la respuesta principal: ${res.statusCode}`);
});
```

101 Upgrade statuses do not fire this event due to their break from the traditional HTTP request/response chain, such as web sockets, in-place TLS upgrades, or HTTP 2.0. To be notified of 101 Upgrade notices, listen for the [`'upgrade'`][] event instead.

### Evento: 'response'

<!-- YAML
added: v0.1.0
-->

* `response` {http.IncomingMessage}

Emitido cuando se recibe una respuesta para esta solicitud. Este evento se emite solo una vez.

### Evento: 'socket'

<!-- YAML
added: v0.5.3
-->

* `socket` {net.Socket}

Emitido luego de que se asigna un socket a esta solicitud.

### Evento: 'timeout'

<!-- YAML
added: v0.7.8
-->

Emitido cuando el socket subyacente agota el tiempo de espera por inactividad. Esto solo notifica que el socket ha estado inactivo. La solicitud debe ser abortada manualmente.

Ver también: [`request.setTimeout()`][].

### Evento: 'upgrade'

<!-- YAML
added: v0.1.94
-->

* `response` {http.IncomingMessage}
* `socket` {net.Socket}
* `head` {Buffer}

Emitido cada vez que un servidor responde a una solicitud con un upgrade. If this event is not being listened for and the response status code is 101 Switching Protocols, clients receiving an upgrade header will have their connections closed.

A client server pair demonstrating how to listen for the `'upgrade'` event.

```js
const http = require('http');

// Create an HTTP server
const srv = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('okay');
});
srv.on('upgrade', (req, socket, head) => {
  socket.write('HTTP/1.1 101 Web Socket Protocol Handshake\r\n' +
               'Upgrade: WebSocket\r\n' +
               'Connection: Upgrade\r\n' +
               '\r\n');

  socket.pipe(socket); // echo back
});

// now that server is running
srv.listen(1337, '127.0.0.1', () => {

  // make a request
  const options = {
    port: 1337,
    hostname: '127.0.0.1',
    headers: {
      'Connection': 'Upgrade',
      'Upgrade': 'websocket'
    }
  };

  const req = http.request(options);
  req.end();

  req.on('upgrade', (res, socket, upgradeHead) => {
    console.log('got upgraded!');
    socket.end();
    process.exit(0);
  });
});
```

### request.abort()

<!-- YAML
added: v0.3.8
-->

Marks the request as aborting. Calling this will cause remaining data in the response to be dropped and the socket to be destroyed.

### request.aborted

<!-- YAML
added: v0.11.14
-->

If a request has been aborted, this value is the time when the request was aborted, in milliseconds since 1 January 1970 00:00:00 UTC.

### request.connection

<!-- YAML
added: v0.3.0
-->

* {net.Socket}

See [`request.socket`][].

### request.end(\[data[, encoding]\]\[, callback\])

<!-- YAML
added: v0.1.90
changes:

  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18780
    description: This method now returns a reference to `ClientRequest`.
-->

* `data` {string|Buffer}
* `encoding` {string}
* `callback` {Function}
* Returns: {this}

Finishes sending the request. If any parts of the body are unsent, it will flush them to the stream. If the request is chunked, this will send the terminating `'0\r\n\r\n'`.

If `data` is specified, it is equivalent to calling [`request.write(data, encoding)`][] followed by `request.end(callback)`.

If `callback` is specified, it will be called when the request stream is finished.

### request.flushHeaders()

<!-- YAML
added: v1.6.0
-->

Flush the request headers.

For efficiency reasons, Node.js normally buffers the request headers until `request.end()` is called or the first chunk of request data is written. It then tries to pack the request headers and data into a single TCP packet.

That's usually desired (it saves a TCP round-trip), but not when the first data is not sent until possibly much later. `request.flushHeaders()` bypasses the optimization and kickstarts the request.

### request.getHeader(name)

<!-- YAML
added: v1.6.0
-->

* `name` {string}
* Returns: {any}

Reads out a header on the request. Note that the name is case insensitive. The type of the return value depends on the arguments provided to [`request.setHeader()`][].

Example:

```js
request.setHeader('content-type', 'text/html');
request.setHeader('Content-Length', Buffer.byteLength(body));
request.setHeader('Set-Cookie', ['type=ninja', 'language=javascript']);
const contentType = request.getHeader('Content-Type');
// contentType is 'text/html'
const contentLength = request.getHeader('Content-Length');
// contentLength is of type number
const setCookie = request.getHeader('set-cookie');
// setCookie is of type string[]
```

### request.maxHeadersCount

* {number} **Default:** `2000`

Limits maximum response headers count. If set to 0, no limit will be applied.

### request.removeHeader(name)

<!-- YAML
added: v1.6.0
-->

* `name` {string}

Removes a header that's already defined into headers object.

Example:

```js
request.removeHeader('Content-Type');
```

### request.setHeader(name, value)

<!-- YAML
added: v1.6.0
-->

* `name` {string}
* `value` {any}

Sets a single header value for headers object. If this header already exists in the to-be-sent headers, its value will be replaced. Use an array of strings here to send multiple headers with the same name. Non-string values will be stored without modification. Therefore, [`request.getHeader()`][] may return non-string values. However, the non-string values will be converted to strings for network transmission.

Example:

```js
request.setHeader('Content-Type', 'application/json');
```

or

```js
request.setHeader('Set-Cookie', ['type=ninja', 'language=javascript']);
```

### request.setNoDelay([noDelay])

<!-- YAML
added: v0.5.9
-->

* `noDelay` {boolean}

Once a socket is assigned to this request and is connected [`socket.setNoDelay()`][] will be called.

### request.setSocketKeepAlive(\[enable\]\[, initialDelay\])

<!-- YAML
added: v0.5.9
-->

* `enable` {boolean}
* `initialDelay` {number}

Once a socket is assigned to this request and is connected [`socket.setKeepAlive()`][] will be called.

### request.setTimeout(timeout[, callback])

<!-- YAML
added: v0.5.9
-->

* `timeout` {number} Milliseconds before a request times out.
* `callback` {Function} Optional function to be called when a timeout occurs. Same as binding to the `'timeout'` event.
* Returns: {http.ClientRequest}

Once a socket is assigned to this request and is connected [`socket.setTimeout()`][] will be called.

### request.socket

<!-- YAML
added: v0.3.0
-->

* {net.Socket}

Reference to the underlying socket. Usually users will not want to access this property. In particular, the socket will not emit `'readable'` events because of how the protocol parser attaches to the socket. The `socket` may also be accessed via `request.connection`.

Example:

```js
const http = require('http');
const options = {
  host: 'www.google.com',
};
const req = http.get(options);
req.end();
req.once('response', (res) => {
  const ip = req.socket.localAddress;
  const port = req.socket.localPort;
  console.log(`Your IP address is ${ip} and your source port is ${port}.`);
  // consume response object
});
```

### request.write(chunk\[, encoding\]\[, callback\])

<!-- YAML
added: v0.1.29
-->

* `chunk` {string|Buffer}
* `encoding` {string}
* `callback` {Function}
* Returns: {boolean}

Sends a chunk of the body. By calling this method many times, a request body can be sent to a server — in that case it is suggested to use the `['Transfer-Encoding', 'chunked']` header line when creating the request.

The `encoding` argument is optional and only applies when `chunk` is a string. Defaults to `'utf8'`.

The `callback` argument is optional and will be called when this chunk of data is flushed.

Returns `true` if the entire data was flushed successfully to the kernel buffer. Returns `false` if all or part of the data was queued in user memory. `'drain'` will be emitted when the buffer is free again.

## Class: http.Server

<!-- YAML
added: v0.1.17
-->

This class inherits from [`net.Server`][] and has the following additional events:

### Event: 'checkContinue'

<!-- YAML
added: v0.3.0
-->

* `request` {http.IncomingMessage}
* `response` {http.ServerResponse}

Emitted each time a request with an HTTP `Expect: 100-continue` is received. If this event is not listened for, the server will automatically respond with a `100 Continue` as appropriate.

Handling this event involves calling [`response.writeContinue()`][] if the client should continue to send the request body, or generating an appropriate HTTP response (e.g. 400 Bad Request) if the client should not continue to send the request body.

Note that when this event is emitted and handled, the [`'request'`][] event will not be emitted.

### Event: 'checkExpectation'

<!-- YAML
added: v5.5.0
-->

* `request` {http.IncomingMessage}
* `response` {http.ServerResponse}

Emitted each time a request with an HTTP `Expect` header is received, where the value is not `100-continue`. If this event is not listened for, the server will automatically respond with a `417 Expectation Failed` as appropriate.

Note that when this event is emitted and handled, the [`'request'`][] event will not be emitted.

### Event: 'clientError'

<!-- YAML
added: v0.1.94
changes:

  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/4557
    description: The default action of calling `.destroy()` on the `socket`
                 will no longer take place if there are listeners attached
                 for `'clientError'`.
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/17672
    description: The `rawPacket` is the current buffer that just parsed. Adding
                 this buffer to the error object of `'clientError'` event is to
                 make it possible that developers can log the broken packet.
-->

* `exception` {Error}
* `socket` {net.Socket}

If a client connection emits an `'error'` event, it will be forwarded here. Listener of this event is responsible for closing/destroying the underlying socket. For example, one may wish to more gracefully close the socket with a custom HTTP response instead of abruptly severing the connection.

Default behavior is to close the socket with an HTTP '400 Bad Request' response if possible, otherwise the socket is immediately destroyed.

`socket` is the [`net.Socket`][] object that the error originated from.

```js
const http = require('http');

const server = http.createServer((req, res) => {
  res.end();
});
server.on('clientError', (err, socket) => {
  socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
});
server.listen(8000);
```

When the `'clientError'` event occurs, there is no `request` or `response` object, so any HTTP response sent, including response headers and payload, *must* be written directly to the `socket` object. Care must be taken to ensure the response is a properly formatted HTTP response message.

`err` is an instance of `Error` with two extra columns:

* `bytesParsed`: the bytes count of request packet that Node.js may have parsed correctly;
* `rawPacket`: the raw packet of current request.

### Event: 'close'

<!-- YAML
added: v0.1.4
-->

Emitted when the server closes.

### Event: 'connect'

<!-- YAML
added: v0.7.0
-->

* `request` {http.IncomingMessage} Arguments for the HTTP request, as it is in the [`'request'`][] event
* `socket` {net.Socket} Network socket between the server and client
* `head` {Buffer} The first packet of the tunneling stream (may be empty)

Emitted each time a client requests an HTTP `CONNECT` method. If this event is not listened for, then clients requesting a `CONNECT` method will have their connections closed.

After this event is emitted, the request's socket will not have a `'data'` event listener, meaning it will need to be bound in order to handle data sent to the server on that socket.

### Event: 'connection'

<!-- YAML
added: v0.1.0
-->

* `socket` {net.Socket}

This event is emitted when a new TCP stream is established. `socket` is typically an object of type [`net.Socket`][]. Usually users will not want to access this event. In particular, the socket will not emit `'readable'` events because of how the protocol parser attaches to the socket. The `socket` can also be accessed at `request.connection`.

This event can also be explicitly emitted by users to inject connections into the HTTP server. In that case, any [`Duplex`][] stream can be passed.

### Event: 'request'

<!-- YAML
added: v0.1.0
-->

* `request` {http.IncomingMessage}
* `response` {http.ServerResponse}

Emitted each time there is a request. Note that there may be multiple requests per connection (in the case of HTTP Keep-Alive connections).

### Event: 'upgrade'

<!-- YAML
added: v0.1.94
changes:

  - version: v10.0.0
    pr-url: v10.0.0
    description: Not listening to this event no longer causes the socket
                 to be destroyed if a client sends an Upgrade header.
-->

* `request` {http.IncomingMessage} Arguments for the HTTP request, as it is in the [`'request'`][] event
* `socket` {net.Socket} Network socket between the server and client
* `head` {Buffer} The first packet of the upgraded stream (may be empty)

Emitted each time a client requests an HTTP upgrade. Listening to this event is optional and clients cannot insist on a protocol change.

After this event is emitted, the request's socket will not have a `'data'` event listener, meaning it will need to be bound in order to handle data sent to the server on that socket.

### server.close([callback])

<!-- YAML
added: v0.1.90
-->

* `callback` {Function}

Stops the server from accepting new connections. See [`net.Server.close()`][].

### server.listen()

Starts the HTTP server listening for connections. This method is identical to [`server.listen()`][] from [`net.Server`][].

### server.listening

<!-- YAML
added: v5.7.0
-->

* {boolean} Indicates whether or not the server is listening for connections.

### server.maxHeadersCount

<!-- YAML
added: v0.7.0
-->

* {number} **Default:** `2000`

Limits maximum incoming headers count. If set to 0, no limit will be applied.

### server.setTimeout(\[msecs\]\[, callback\])

<!-- YAML
added: v0.9.12
-->

* `msecs` {number} **Default:** `120000` (2 minutes)
* `callback` {Function}
* Returns: {http.Server}

Sets the timeout value for sockets, and emits a `'timeout'` event on the Server object, passing the socket as an argument, if a timeout occurs.

If there is a `'timeout'` event listener on the Server object, then it will be called with the timed-out socket as an argument.

By default, the Server's timeout value is 2 minutes, and sockets are destroyed automatically if they time out. However, if a callback is assigned to the Server's `'timeout'` event, timeouts must be handled explicitly.

### server.timeout

<!-- YAML
added: v0.9.12
-->

* {number} Timeout in milliseconds. **Default:** `120000` (2 minutes).

The number of milliseconds of inactivity before a socket is presumed to have timed out.

A value of `0` will disable the timeout behavior on incoming connections.

The socket timeout logic is set up on connection, so changing this value only affects new connections to the server, not any existing connections.

### server.keepAliveTimeout

<!-- YAML
added: v8.0.0
-->

* {number} Timeout in milliseconds. **Default:** `5000` (5 seconds).

The number of milliseconds of inactivity a server needs to wait for additional incoming data, after it has finished writing the last response, before a socket will be destroyed. If the server receives new data before the keep-alive timeout has fired, it will reset the regular inactivity timeout, i.e., [`server.timeout`][].

A value of `0` will disable the keep-alive timeout behavior on incoming connections. A value of `0` makes the http server behave similarly to Node.js versions prior to 8.0.0, which did not have a keep-alive timeout.

The socket timeout logic is set up on connection, so changing this value only affects new connections to the server, not any existing connections.

## Class: http.ServerResponse

<!-- YAML
added: v0.1.17
-->

This object is created internally by an HTTP server — not by the user. It is passed as the second parameter to the [`'request'`][] event.

The response implements, but does not inherit from, the [Writable Stream](stream.html#stream_class_stream_writable) interface. This is an [`EventEmitter`][] with the following events:

### Event: 'close'

<!-- YAML
added: v0.6.7
-->

Indicates that the underlying connection was terminated before [`response.end()`][] was called or able to flush.

### Event: 'finish'

<!-- YAML
added: v0.3.6
-->

Emitted when the response has been sent. More specifically, this event is emitted when the last segment of the response headers and body have been handed off to the operating system for transmission over the network. It does not imply that the client has received anything yet.

After this event, no more events will be emitted on the response object.

### response.addTrailers(headers)

<!-- YAML
added: v0.3.0
-->

* `headers` {Object}

This method adds HTTP trailing headers (a header but at the end of the message) to the response.

Trailers will **only** be emitted if chunked encoding is used for the response; if it is not (e.g. if the request was HTTP/1.0), they will be silently discarded.

Note that HTTP requires the `Trailer` header to be sent in order to emit trailers, with a list of the header fields in its value. E.g.,

```js
response.writeHead(200, { 'Content-Type': 'text/plain',
                          'Trailer': 'Content-MD5' });
response.write(fileData);
response.addTrailers({ 'Content-MD5': '7895bf4b8828b55ceaf47747b4bca667' });
response.end();
```

Attempting to set a header field name or value that contains invalid characters will result in a [`TypeError`][] being thrown.

### response.connection

<!-- YAML
added: v0.3.0
-->

* {net.Socket}

See [`response.socket`][].

### response.end(\[data\]\[, encoding\][, callback])

<!-- YAML
added: v0.1.90
changes:

  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18780
    description: This method now returns a reference to `ServerResponse`.
-->

* `data` {string|Buffer}
* `encoding` {string}
* `callback` {Function}
* Returns: {this}

This method signals to the server that all of the response headers and body have been sent; that server should consider this message complete. The method, `response.end()`, MUST be called on each response.

If `data` is specified, it is equivalent to calling [`response.write(data, encoding)`][] followed by `response.end(callback)`.

If `callback` is specified, it will be called when the response stream is finished.

### response.finished

<!-- YAML
added: v0.0.2
-->

* {boolean}

Boolean value that indicates whether the response has completed. Starts as `false`. After [`response.end()`][] executes, the value will be `true`.

### response.getHeader(name)

<!-- YAML
added: v0.4.0
-->

* `name` {string}
* Returns: {any}

Reads out a header that's already been queued but not sent to the client. Note that the name is case insensitive. The type of the return value depends on the arguments provided to [`response.setHeader()`][].

Example:

```js
response.setHeader('Content-Type', 'text/html');
response.setHeader('Content-Length', Buffer.byteLength(body));
response.setHeader('Set-Cookie', ['type=ninja', 'language=javascript']);
const contentType = response.getHeader('content-type');
// contentType is 'text/html'
const contentLength = response.getHeader('Content-Length');
// contentLength is of type number
const setCookie = response.getHeader('set-cookie');
// setCookie is of type string[]
```

### response.getHeaderNames()

<!-- YAML
added: v7.7.0
-->

* Returns: {string[]}

Returns an array containing the unique names of the current outgoing headers. All header names are lowercase.

Example:

```js
response.setHeader('Foo', 'bar');
response.setHeader('Set-Cookie', ['foo=bar', 'bar=baz']);

const headerNames = response.getHeaderNames();
// headerNames === ['foo', 'set-cookie']
```

### response.getHeaders()

<!-- YAML
added: v7.7.0
-->

* Returns: {Object}

Returns a shallow copy of the current outgoing headers. Since a shallow copy is used, array values may be mutated without additional calls to various header-related http module methods. The keys of the returned object are the header names and the values are the respective header values. All header names are lowercase.

The object returned by the `response.getHeaders()` method *does not* prototypically inherit from the JavaScript `Object`. This means that typical `Object` methods such as `obj.toString()`, `obj.hasOwnProperty()`, and others are not defined and *will not work*.

Example:

```js
response.setHeader('Foo', 'bar');
response.setHeader('Set-Cookie', ['foo=bar', 'bar=baz']);

const headers = response.getHeaders();
// headers === { foo: 'bar', 'set-cookie': ['foo=bar', 'bar=baz'] }
```

### response.hasHeader(name)

<!-- YAML
added: v7.7.0
-->

* `name` {string}
* Returns: {boolean}

Returns `true` if the header identified by `name` is currently set in the outgoing headers. Note that the header name matching is case-insensitive.

Example:

```js
const hasContentType = response.hasHeader('content-type');
```

### response.headersSent

<!-- YAML
added: v0.9.3
-->

* {boolean}

Boolean (read-only). True if headers were sent, false otherwise.

### response.removeHeader(name)

<!-- YAML
added: v0.4.0
-->

* `name` {string}

Removes a header that's queued for implicit sending.

Example:

```js
response.removeHeader('Content-Encoding');
```

### response.sendDate

<!-- YAML
added: v0.7.5
-->

* {boolean}

When true, the Date header will be automatically generated and sent in the response if it is not already present in the headers. Defaults to true.

This should only be disabled for testing; HTTP requires the Date header in responses.

### response.setHeader(name, value)

<!-- YAML
added: v0.4.0
-->

* `name` {string}
* `value` {any}

Sets a single header value for implicit headers. If this header already exists in the to-be-sent headers, its value will be replaced. Use an array of strings here to send multiple headers with the same name. Non-string values will be stored without modification. Therefore, [`response.getHeader()`][] may return non-string values. However, the non-string values will be converted to strings for network transmission.

Example:

```js
response.setHeader('Content-Type', 'text/html');
```

or

```js
response.setHeader('Set-Cookie', ['type=ninja', 'language=javascript']);
```

Attempting to set a header field name or value that contains invalid characters will result in a [`TypeError`][] being thrown.

When headers have been set with [`response.setHeader()`][], they will be merged with any headers passed to [`response.writeHead()`][], with the headers passed to [`response.writeHead()`][] given precedence.

```js
// returns content-type = text/plain
const server = http.createServer((req, res) => {
  res.setHeader('Content-Type', 'text/html');
  res.setHeader('X-Foo', 'bar');
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('ok');
});
```

### response.setTimeout(msecs[, callback])

<!-- YAML
added: v0.9.12
-->

* `msecs` {number}
* `callback` {Function}
* Returns: {http.ServerResponse}

Sets the Socket's timeout value to `msecs`. If a callback is provided, then it is added as a listener on the `'timeout'` event on the response object.

If no `'timeout'` listener is added to the request, the response, or the server, then sockets are destroyed when they time out. If a handler is assigned to the request, the response, or the server's `'timeout'` events, timed out sockets must be handled explicitly.

### response.socket

<!-- YAML
added: v0.3.0
-->

* {net.Socket}

Reference to the underlying socket. Usually users will not want to access this property. In particular, the socket will not emit `'readable'` events because of how the protocol parser attaches to the socket. After `response.end()`, the property is nulled. The `socket` may also be accessed via `response.connection`.

Example:

```js
const http = require('http');
const server = http.createServer((req, res) => {
  const ip = res.socket.remoteAddress;
  const port = res.socket.remotePort;
  res.end(`Your IP address is ${ip} and your source port is ${port}.`);
}).listen(3000);
```

### response.statusCode

<!-- YAML
added: v0.4.0
-->

* {number}

When using implicit headers (not calling [`response.writeHead()`][] explicitly), this property controls the status code that will be sent to the client when the headers get flushed.

Example:

```js
response.statusCode = 404;
```

After response header was sent to the client, this property indicates the status code which was sent out.

### response.statusMessage

<!-- YAML
added: v0.11.8
-->

* {string}

When using implicit headers (not calling [`response.writeHead()`][] explicitly), this property controls the status message that will be sent to the client when the headers get flushed. If this is left as `undefined` then the standard message for the status code will be used.

Example:

```js
response.statusMessage = 'Not found';
```

After response header was sent to the client, this property indicates the status message which was sent out.

### response.write(chunk\[, encoding\]\[, callback\])

<!-- YAML
added: v0.1.29
-->

* `chunk` {string|Buffer}
* `encoding` {string} **Default:** `'utf8'`
* `callback` {Function}
* Returns: {boolean}

If this method is called and [`response.writeHead()`][] has not been called, it will switch to implicit header mode and flush the implicit headers.

This sends a chunk of the response body. This method may be called multiple times to provide successive parts of the body.

Note that in the `http` module, the response body is omitted when the request is a HEAD request. Similarly, the `204` and `304` responses *must not* include a message body.

`chunk` can be a string or a buffer. If `chunk` is a string, the second parameter specifies how to encode it into a byte stream. `callback` will be called when this chunk of data is flushed.

This is the raw HTTP body and has nothing to do with higher-level multi-part body encodings that may be used.

The first time [`response.write()`][] is called, it will send the buffered header information and the first chunk of the body to the client. The second time [`response.write()`][] is called, Node.js assumes data will be streamed, and sends the new data separately. That is, the response is buffered up to the first chunk of the body.

Returns `true` if the entire data was flushed successfully to the kernel buffer. Returns `false` if all or part of the data was queued in user memory. `'drain'` will be emitted when the buffer is free again.

### response.writeContinue()

<!-- YAML
added: v0.3.0
-->

Sends a HTTP/1.1 100 Continue message to the client, indicating that the request body should be sent. See the [`'checkContinue'`][] event on `Server`.

### response.writeHead(statusCode\[, statusMessage\]\[, headers\])

<!-- YAML
added: v0.1.30
changes:

  - version: v5.11.0, v4.4.5
    pr-url: https://github.com/nodejs/node/pull/6291
    description: A `RangeError` is thrown if `statusCode` is not a number in
                 the range `[100, 999]`.
-->

* `statusCode` {number}
* `statusMessage` {string}
* `headers` {Object}

Sends a response header to the request. The status code is a 3-digit HTTP status code, like `404`. The last argument, `headers`, are the response headers. Optionally one can give a human-readable `statusMessage` as the second argument.

Example:

```js
const body = 'hello world';
response.writeHead(200, {
  'Content-Length': Buffer.byteLength(body),
  'Content-Type': 'text/plain' });
```

This method must only be called once on a message and it must be called before [`response.end()`][] is called.

If [`response.write()`][] or [`response.end()`][] are called before calling this, the implicit/mutable headers will be calculated and call this function.

When headers have been set with [`response.setHeader()`][], they will be merged with any headers passed to [`response.writeHead()`][], with the headers passed to [`response.writeHead()`][] given precedence.

```js
// returns content-type = text/plain
const server = http.createServer((req, res) => {
  res.setHeader('Content-Type', 'text/html');
  res.setHeader('X-Foo', 'bar');
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('ok');
});
```

Note that Content-Length is given in bytes not characters. The above example works because the string `'hello world'` contains only single byte characters. If the body contains higher coded characters then `Buffer.byteLength()` should be used to determine the number of bytes in a given encoding. And Node.js does not check whether Content-Length and the length of the body which has been transmitted are equal or not.

Attempting to set a header field name or value that contains invalid characters will result in a [`TypeError`][] being thrown.

### response.writeProcessing()

<!-- YAML
added: v10.0.0
-->

Sends a HTTP/1.1 102 Processing message to the client, indicating that the request body should be sent.

## Class: http.IncomingMessage

<!-- YAML
added: v0.1.17
-->

An `IncomingMessage` object is created by [`http.Server`][] or [`http.ClientRequest`][] and passed as the first argument to the [`'request'`][] and [`'response'`][] event respectively. It may be used to access response status, headers and data.

It implements the [Readable Stream](stream.html#stream_class_stream_readable) interface, as well as the following additional events, methods, and properties.

### Event: 'aborted'

<!-- YAML
added: v0.3.8
-->

Emitted when the request has been aborted.

### Event: 'close'

<!-- YAML
added: v0.4.2
-->

Indicates that the underlying connection was closed. Just like `'end'`, this event occurs only once per response.

### message.aborted

<!-- YAML
added: v10.1.0
-->

* {boolean}

The `message.aborted` property will be `true` if the request has been aborted.

### message.destroy([error])

<!-- YAML
added: v0.3.0
-->

* `error` {Error}

Calls `destroy()` on the socket that received the `IncomingMessage`. If `error` is provided, an `'error'` event is emitted and `error` is passed as an argument to any listeners on the event.

### message.headers

<!-- YAML
added: v0.1.5
-->

* {Object}

The request/response headers object.

Key-value pairs of header names and values. Header names are lower-cased. Example:

```js
// Prints something like:
//
// { 'user-agent': 'curl/7.22.0',
//   host: '127.0.0.1:8000',
//   accept: '*/*' }
console.log(request.headers);
```

Duplicates in raw headers are handled in the following ways, depending on the header name:

* Duplicates of `age`, `authorization`, `content-length`, `content-type`, `etag`, `expires`, `from`, `host`, `if-modified-since`, `if-unmodified-since`, `last-modified`, `location`, `max-forwards`, `proxy-authorization`, `referer`, `retry-after`, or `user-agent` are discarded.
* `set-cookie` is always an array. Duplicates are added to the array.
* For all other headers, the values are joined together with ', '.

### message.httpVersion

<!-- YAML
added: v0.1.1
-->

* {string}

In case of server request, the HTTP version sent by the client. In the case of client response, the HTTP version of the connected-to server. Probably either `'1.1'` or `'1.0'`.

Also `message.httpVersionMajor` is the first integer and `message.httpVersionMinor` is the second.

### message.method

<!-- YAML
added: v0.1.1
-->

* {string}

**Only valid for request obtained from [`http.Server`][].**

The request method as a string. Read only. Example: `'GET'`, `'DELETE'`.

### message.rawHeaders

<!-- YAML
added: v0.11.6
-->

* {string[]}

The raw request/response headers list exactly as they were received.

Note that the keys and values are in the same list. It is *not* a list of tuples. So, the even-numbered offsets are key values, and the odd-numbered offsets are the associated values.

Header names are not lowercased, and duplicates are not merged.

```js
// Prints something like:
//
// [ 'user-agent',
//   'this is invalid because there can be only one',
//   'User-Agent',
//   'curl/7.22.0',
//   'Host',
//   '127.0.0.1:8000',
//   'ACCEPT',
//   '*/*' ]
console.log(request.rawHeaders);
```

### message.rawTrailers

<!-- YAML
added: v0.11.6
-->

* {string[]}

The raw request/response trailer keys and values exactly as they were received. Only populated at the `'end'` event.

### message.setTimeout(msecs, callback)

<!-- YAML
added: v0.5.9
-->

* `msecs` {number}
* `callback` {Function}
* Returns: {http.IncomingMessage}

Calls `message.connection.setTimeout(msecs, callback)`.

### message.socket

<!-- YAML
added: v0.3.0
-->

* {net.Socket}

The [`net.Socket`][] object associated with the connection.

With HTTPS support, use [`request.socket.getPeerCertificate()`][] to obtain the client's authentication details.

### message.statusCode

<!-- YAML
added: v0.1.1
-->

* {number}

**Only valid for response obtained from [`http.ClientRequest`][].**

The 3-digit HTTP response status code. E.G. `404`.

### message.statusMessage

<!-- YAML
added: v0.11.10
-->

* {string}

**Only valid for response obtained from [`http.ClientRequest`][].**

The HTTP response status message (reason phrase). E.G. `OK` or `Internal Server
Error`.

### message.trailers

<!-- YAML
added: v0.3.0
-->

* {Object}

The request/response trailers object. Only populated at the `'end'` event.

### message.url

<!-- YAML
added: v0.1.90
-->

* {string}

**Only valid for request obtained from [`http.Server`][].**

Request URL string. This contains only the URL that is present in the actual HTTP request. If the request is:

```txt
GET /status?name=ryan HTTP/1.1\r\n
Accept: text/plain\r\n
\r\n
```

Then `request.url` will be:

<!-- eslint-disable semi -->

```js
'/status?name=ryan'
```

To parse the url into its parts `require('url').parse(request.url)` can be used. Example:

```txt
$ node
> require('url').parse('/status?name=ryan')
Url {
  protocol: null,
  slashes: null,
  auth: null,
  host: null,
  port: null,
  hostname: null,
  hash: null,
  search: '?name=ryan',
  query: 'name=ryan',
  pathname: '/status',
  path: '/status?name=ryan',
  href: '/status?name=ryan' }
```

To extract the parameters from the query string, the `require('querystring').parse` function can be used, or `true` can be passed as the second argument to `require('url').parse`. Example:

```txt
$ node
> require('url').parse('/status?name=ryan', true)
Url {
  protocol: null,
  slashes: null,
  auth: null,
  host: null,
  port: null,
  hostname: null,
  hash: null,
  search: '?name=ryan',
  query: { name: 'ryan' },
  pathname: '/status',
  path: '/status?name=ryan',
  href: '/status?name=ryan' }
```

## http.METHODS

<!-- YAML
added: v0.11.8
-->

* {string[]}

A list of the HTTP methods that are supported by the parser.

## http.STATUS_CODES

<!-- YAML
added: v0.1.22
-->

* {Object}

A collection of all the standard HTTP response status codes, and the short description of each. For example, `http.STATUS_CODES[404] === 'Not
Found'`.

## http.createServer(\[options\]\[, requestListener\])

<!-- YAML
added: v0.1.13
changes:

  - version: v9.6.0
    pr-url: https://github.com/nodejs/node/pull/15752
    description: The `options` argument is supported now.
-->

* `options` {Object} 
  * `IncomingMessage` {http.IncomingMessage} Specifies the `IncomingMessage` class to be used. Useful for extending the original `IncomingMessage`. **Default:** `IncomingMessage`.
  * `ServerResponse` {http.ServerResponse} Specifies the `ServerResponse` class to be used. Useful for extending the original `ServerResponse`. **Default:** `ServerResponse`.

* `requestListener` {Function}

* Returns: {http.Server}

Returns a new instance of [`http.Server`][].

The `requestListener` is a function which is automatically added to the [`'request'`][] event.

## http.get(options[, callback])

<!-- YAML
added: v0.3.6
changes:

  - version: v7.5.0
    pr-url: https://github.com/nodejs/node/pull/10638
    description: The `options` parameter can be a WHATWG `URL` object.
-->

* `options` {Object | string | URL} Accepts the same `options` as [`http.request()`][], with the `method` always set to `GET`. Properties that are inherited from the prototype are ignored.
* `callback` {Function}
* Returns: {http.ClientRequest}

Since most requests are GET requests without bodies, Node.js provides this convenience method. The only difference between this method and [`http.request()`][] is that it sets the method to GET and calls `req.end()` automatically. Note that the callback must take care to consume the response data for reasons stated in [`http.ClientRequest`][] section.

The `callback` is invoked with a single argument that is an instance of [`http.IncomingMessage`][].

JSON Fetching Example:

```js
http.get('http://nodejs.org/dist/index.json', (res) => {
  const { statusCode } = res;
  const contentType = res.headers['content-type'];

  let error;
  if (statusCode !== 200) {
    error = new Error('Request Failed.\n' +
                      `Status Code: ${statusCode}`);
  } else if (!/^application\/json/.test(contentType)) {
    error = new Error('Invalid content-type.\n' +
                      `Expected application/json but received ${contentType}`);
  }
  if (error) {
    console.error(error.message);
    // consume response data to free up memory
    res.resume();
    return;
  }

  res.setEncoding('utf8');
  let rawData = '';
  res.on('data', (chunk) => { rawData += chunk; });
  res.on('end', () => {
    try {
      const parsedData = JSON.parse(rawData);
      console.log(parsedData);
    } catch (e) {
      console.error(e.message);
    }
  });
}).on('error', (e) => {
  console.error(`Got error: ${e.message}`);
});
```

## http.globalAgent

<!-- YAML
added: v0.5.9
-->

* {http.Agent}

Global instance of `Agent` which is used as the default for all HTTP client requests.

## http.request(options[, callback])

<!-- YAML
added: v0.3.6
changes:

  - version: v7.5.0
    pr-url: https://github.com/nodejs/node/pull/10638
    description: The `options` parameter can be a WHATWG `URL` object.
-->

* `options` {Object | string | URL} 
  * `protocol` {string} Protocol to use. **Default:** `'http:'`.
  * `host` {string} A domain name or IP address of the server to issue the request to. **Default:** `'localhost'`.
  * `hostname` {string} Alias for `host`. To support [`url.parse()`][], `hostname` is preferred over `host`.
  * `family` {number} IP address family to use when resolving `host` and `hostname`. Valid values are `4` or `6`. When unspecified, both IP v4 and v6 will be used.
  * `port` {number} Port of remote server. **Default:** `80`.
  * `localAddress` {string} Local interface to bind for network connections.
  * `socketPath` {string} Unix Domain Socket (use one of `host:port` or `socketPath`).
  * `method` {string} A string specifying the HTTP request method. **Default:** `'GET'`.
  * `path` {string} Request path. Should include query string if any. E.G. `'/index.html?page=12'`. An exception is thrown when the request path contains illegal characters. Currently, only spaces are rejected but that may change in the future. **Default:** `'/'`.
  * `headers` {Object} An object containing request headers.
  * `auth` {string} Basic authentication i.e. `'user:password'` to compute an Authorization header.
  * `agent` {http.Agent | boolean} Controls [`Agent`][] behavior. Possible values: 
    * `undefined` (default): use [`http.globalAgent`][] for this host and port.
    * `Agent` object: explicitly use the passed in `Agent`.
    * `false`: causes a new `Agent` with default values to be used.
  * `createConnection` {Function} A function that produces a socket/stream to use for the request when the `agent` option is not used. This can be used to avoid creating a custom `Agent` class just to override the default `createConnection` function. See [`agent.createConnection()`][] for more details. Any [`Duplex`][] stream is a valid return value.
  * `timeout` {number}: A number specifying the socket timeout in milliseconds. This will set the timeout before the socket is connected.
  * `setHost` {boolean}: Specifies whether or not to automatically add the `Host` header. Defaults to `true`.
* `callback` {Function}
* Returns: {http.ClientRequest}

Node.js maintains several connections per server to make HTTP requests. This function allows one to transparently issue requests.

`options` can be an object, a string, or a [`URL`][] object. If `options` is a string, it is automatically parsed with [`url.parse()`][]. If it is a [`URL`][] object, it will be automatically converted to an ordinary `options` object.

The optional `callback` parameter will be added as a one-time listener for the [`'response'`][] event.

`http.request()` returns an instance of the [`http.ClientRequest`][] class. The `ClientRequest` instance is a writable stream. If one needs to upload a file with a POST request, then write to the `ClientRequest` object.

Example:

```js
const postData = querystring.stringify({
  'msg': 'Hello World!'
});

const options = {
  hostname: 'www.google.com',
  port: 80,
  path: '/upload',
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'Content-Length': Buffer.byteLength(postData)
  }
};

const req = http.request(options, (res) => {
  console.log(`STATUS: ${res.statusCode}`);
  console.log(`HEADERS: ${JSON.stringify(res.headers)}`);
  res.setEncoding('utf8');
  res.on('data', (chunk) => {
    console.log(`BODY: ${chunk}`);
  });
  res.on('end', () => {
    console.log('No more data in response.');
  });
});

req.on('error', (e) => {
  console.error(`problem with request: ${e.message}`);
});

// write data to request body
req.write(postData);
req.end();
```

Note that in the example `req.end()` was called. With `http.request()` one must always call `req.end()` to signify the end of the request - even if there is no data being written to the request body.

If any error is encountered during the request (be that with DNS resolution, TCP level errors, or actual HTTP parse errors) an `'error'` event is emitted on the returned request object. As with all `'error'` events, if no listeners are registered the error will be thrown.

There are a few special headers that should be noted.

* Sending a 'Connection: keep-alive' will notify Node.js that the connection to the server should be persisted until the next request.

* Sending a 'Content-Length' header will disable the default chunked encoding.

* Sending an 'Expect' header will immediately send the request headers. Usually, when sending 'Expect: 100-continue', both a timeout and a listener for the `'continue'` event should be set. See RFC2616 Section 8.2.3 for more information.

* Sending an Authorization header will override using the `auth` option to compute basic authentication.

Example using a [`URL`][] as `options`:

```js
const options = new URL('http://abc:xyz@example.com');

const req = http.request(options, (res) => {
  // ...
});
```

In a successful request, the following events will be emitted in the following order:

* `'socket'`
* `'response'` 
  * `'data'` any number of times, on the `res` object (`'data'` will not be emitted at all if the response body is empty, for instance, in most redirects)
  * `'end'` on the `res` object
* `'close'`

In the case of a connection error, the following events will be emitted:

* `'socket'`
* `'error'`
* `'close'`

If `req.abort()` is called before the connection succeeds, the following events will be emitted in the following order:

* `'socket'`
* (`req.abort()` called here)
* `'abort'`
* `'close'`
* `'error'` with an error with message `'Error: socket hang up'` and code `'ECONNRESET'`

If `req.abort()` is called after the response is received, the following events will be emitted in the following order:

* `'socket'`
* `'response'` 
  * `'data'` any number of times, on the `res` object
* (`req.abort()` called here)
* `'abort'`
* `'close'` 
  * `'aborted'` on the `res` object
  * `'end'` on the `res` object
  * `'close'` on the `res` object

Note that setting the `timeout` option or using the `setTimeout()` function will not abort the request or do anything besides add a `'timeout'` event.