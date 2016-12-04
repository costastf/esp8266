# Micropython HTTP daemon

This set of modules provides a simple HTTP framework and server, providing an HTTP interface into your ESP8266.  Having an HTTP server allows developers to create management consoles into their ESP8266 devices, which can be very useful for configuration and troubleshooting devices, or even for bootstrapping them from an initial state.

By itself, the `uhttpd` module is just a TCP server and framework for adding handlers to process HTTP requests.  The actual servicing of HTTP requests is done by installing handlers, which are configured in the `uhttpd` server.  It's the handlers that actually do the heavy lifting when it comes to servicing HTTP requests.  

This package includes handlers for servicing files on the micropython file system (e.g., HTML, Javascript, CSS, etc), as well as handlers for managing REST-ful API calls, essential components in any modern web-based application.
 
A driving design goal of this package is to have minimal impact on the ESP8266 device, itself, and to provide the tools that allow developers to implement rich client-side applications.  Be design, web applications built with this framework should do as little work as possible on the server side, but should instead make use of modern web technologies to allow the web client or browser to perform significant parts of business logic.  In most cases, the web clients will have far more memory and compute resources than the ESP8266 device, itself, so it is wise to keep as much logic as possible on the client side.

> Warning: This software provides _no security_ for your applications.  When this software is running, any machine on your network may connect to your ESP8266 and browse the parts of the file system you expose through configuration, including possibly sensitive security credentials stored in plain text.  AS WRITTEN, THIS SOFTWARE IS NOT INTENDED FOR USE IN AN UNTRUSTED NETWORK!

## Modules and Dependencies

The `uhttpd` framework and server is comprised the following python modules:

* `uhttpd.py` -- provides HTTP server and framework
* `utcp_server.py` -- provides basic TCP networking layer of abstraction
* `http_file_handler.py` -- a file handler for the `uhttpd` server
* `http_api_handler.py` -- a handler for servicing REST-ful APIs
* `stats_api_handler` -- a handler instance used to service run-time statistics about the device

This module relies on the `ulog.py` facility, defined in the [logging](/micropython/logging) area of this repository.

There is currently no `upip` support for this package.

## Loading Modules

Some of the modules in this package require significant amounts of RAM to load and run.  While you can run these modules by loading them on the Micropython file system and allowing the runtime to compile the source modules for you, it is recommended to either burn the modules into your formware (a rather involved process, and recommended for production), or to compile to bytcode and load the generated `.mpy` files (recommended for development), which will decrease the memory footprint of your application using these modules.  You can acquire `mpy-cross` tool which will compile micro python source modules to bytecode, by installing and building the micro python source project, as described [here](https://github.com/micropython/micropython/tree/master/esp8266)

This package includes a `make` file (`Makefile`), which you can use to generate `.mpy` files.  Loading the generated bytecode files, instead of the python files, will reduce memory overhead during the development process.

> Note.  The Makefile assumes the presence of `mpy-cross` in your executable path.

For example, to build the bytecode, 

    prompt$ export PATH=/Volumes/case-sensitive/micropython/mpy-cross:$PATH
    prompt$ pwd
    /work/src/github/fadushin/esp8266/micropython
    prompt$ make
    mpy-cross logging/ulog.py
    mpy-cross logging/console_sink.py
    mpy-cross logging/syslog_sink.py
    mpy-cross daemons/utcp_server.py
    mpy-cross daemons/uhttpd.py
    mpy-cross daemons/http_file_handler.py
    mpy-cross daemons/http_api_handler.py
    mpy-cross daemons/stats_api.py

If you have `webrepl` running and `webrepl_cli.py` in your `PATH`, then you can upload the files you need to your device (adjusted of course for the IP address of your ESP8266), as follows:

    prompt$ export PATH=/Volumes/case-sensitive/webrepl:$PATH
    prompt$ for i in logging/*.mpy daemons/*.mpy; do webrepl_cli.py $i 192.168.1.180:.; done

## Basic Usage

Start by creating a directory (e.g., `www`) on your file system in which you can place HTML (or other) files:

    >>> import os
    >>> os.mkdir('www')
    >>> os.chdir('www')
    >>> f = open('index.html', 'w')
    >>> f.write('<html><body>Hello World!</body></html>')
    38
    >>> f.close()
    >>> os.listdir()
    ['index.html']

To run the `uhttpd` server, initialize an instance of the `uhttpd.Server` class with an ordered list of tuples, which map URL prefixes to handlers, and start the server.  When creating a HTTP file handler, you can optionally specify the "root" of the file system from which to serve files. 

For example, to start the server with the file handler rooted off the `/www` path, use the following: 

    >>> import uhttpd
    >>> import http_file_handler
    >>> server = uhttpd.Server([('/', http_file_handler.Handler())])
    >>> server.start()

You should then see some logs printed to the console, indicating that the server is listening for connections:

    2000-01-01T08:09:15.005 [info] esp8266: TCP server started on 192.168.4.1:80
    2000-01-01T08:09:15.005 [info] esp8266: TCP server started on 0.0.0.0:80

You may now connect to your ESP8266 via a web browser or curl and browse your file system, e.g.,

    prompt$ curl -i 'http://192.168.1.180/' 
    HTTP/1.1 200 OK
    Content-Length: 38
    Content-Type: text/html
    
    <html><body>Hello World!</body></html>

## Reference

The following sections describe the components that form the `uhttpd` package in more detail.

### `uhttpd.Server`

The `uhttpd.Server` is simply a container for helper instances.  Its only job is to accept connections from clients, to read and parse HTTP headers, and to read the body of the request, if present.  The server will the dispatch the request to the first handler that matches the path indicated in the HTTP request, and wait for a response.  Once received, the response will be sent back to the caller.

An instance of a `uhttpd.Server` is created using an ordered list of pairs, where the first element of the pair is a path prefix, and the second is a handler index.  When an HTTP request is processed, the server will select the handler that corresponds with the first path prefix which is a prefix of the path in the HTTP request.

For example, given the following construction:

    >>> import uhttpd
    >>> handler1 = ...
    >>> handler2 = ...
    >>> handler3 = ...
    >>> server = uhttpd.Server([
            ('/foo/', handler1),
            ('/gnu', handler2),
            ('/', handler3)
        ])
    >>> server.start()

a request of the form `http://host/foo/bar/` will be handled by `handler1`, whereas a request of the form `http://host/gant/` will be handled by `handler3`.

You may optionally specify a port at construction time.  The default is 80.

Once started, the `uhttpd.Server` will listen asynchronously for connections.  While a connection is not being serviced, the application may proceed to do work (e.g., via the REPL).  Once a request is accepted, the entire request processing, including the time spent in the handlers, is synchronous.

A `uhttpd.Server` may be stopped via the `stop` method.

### `http_file_handler.Handler`

The `http_file_handler.Handler` request handler is designed to service files on the ESP8266 file system, relative to a specified file system root path (e.g., `/www`).

This handler will display the contents of the path specified in the HTTP GET URL, relative to the specified root path.  If path refers to a file on the file system, the file contents are removed.  If the path refers to a directory, and the directory does not contain an `index.html` file, the directory contents are provided as a sequence of hyperlinks.  Otherwise, the request will result in a 404/Not Found HTTP error.

The default root path for the `http_file_handler.Handler` is `/www`.  For example, the following constructor will result in a file handler that expects HTTP artifacts to reside in the `/www` directory of the micropython file system:

    >>> import http_file_handler
    >>> file_handler = http_file_handler.Handler()

Once your handler is created, you can then provide it to the `uhttpd.Server` constructor, providing the path prefix used to locate the handler at request time:

    >>> import uhttpd
    >>> server = uhttpd.Server([
            ('/', file_handler)
        ])

> Important: The path prefix provided to the `uhttpd.Server` constructor is distinct from the root path provided to the `http_file_handler.Handler` constructor.  The former is used only to pick out the handler to process the handler.  The latter is used to locate where, on the file system, to start looking for files and directories to serve.  If the root path is `/www` and the path in the HTTP request is `/foo/bar`, then the `http_file_handler.Handler` will look for `/www/foo/bar` on teh micropython file system.

You may of course specify a root path other than `/www` through the `http_file_handler.Handler` constructor, but the directory must exist, or an error will occur at the time of construction. 

> Warning: If you specify the micropython file system root path (`/`) in the HTTP file handler constructor, you may expose sensitive security information, such as the Webrepl password, through the HTTP interface.  This behavior is strongly discouraged.

This handler only supports HTTP GET requests.  Any other request verb will be rejected.

This handler recognizes HTML (`text/html`), CSS (`text/css`), and Javascript (`text/javascript`) file endings, and will set the `content-type` header in the response, accordingly.  The `content-length` header will contain the length of the body.  Any file other than the above list of types is treated as `text/plain`

> Note. Future versions of this handler may support configuration to allow better protection of the file and directory contents, in the spirit of Apache httpd.

### `http_api_handler.Handler`

The `http_api_handler.Handler` request handler is designed to handle REST-ful API calls into the `uhttpd` server.  Currently, JSON is the only supported message binding for REST-ful API calls through this handler.

This handler should be initialized with an ordered list of tuples, mapping a list of API "components" to an API handler instance, which will be used to actually service the API request.  A component, in this sense, is a sequence of path elements

    >>> import http_api_handler
    >>> api1 = ...
    >>> api2 = ...
    >>> api3 = ...
    >>> api_handler = http_api_handler.Handler([
            (['foo'], api1),
            (['gnu'], api2),
            ([], api3),
       ])

You can then add the API handler to the `uhttpd.Server`, as we did above with the HTTP File Handler:

    >>> import uhttpd
    >>> server = uhttpd.Server([
            ('/api', api_handler),
            ('/', file_handler)
        ])

This way, any HTTP requests under `http://host/api` get directed to the HTTP API Handler, and everything else gets directed to the HTTP File Handler.

The HTTP API Handler, like the `uhttp.Server`, does not do much processing on the request, but instead uses the HTTP path to locate the first API Handler that matches the sequence of components provided in the constructor.  In the above example, a request to `http://host/api/foo/` would get processed by `api1` (as would requests to `http://host/api/foo/bar`), whereas requests simply to `http://host/api/` would get procecced by `api3`.

### `stats_api.Handler`

This package includes one HTTP API Handler instance, which can be used to retrieve runtime statistics from the ESP8266 device.  This API is largely for demonstration purposes, but it can also be used as part of an effort to build a Web UI to manage an ESP8266 device.

Here is a complete example of construction of a server using this handler:

    >>> import http_file_handler
    >>> file_handler = http_file_handler.Handler()
    >>> import stats_api
    >>> stats = stats_api.Handler()
    >>> import http_api_handler
    >>> api_handler = http_api_handler.Handler([
            (['stats'], stats)
        ])
    >>> import uhttpd
    >>> server = uhttpd.Server([
            ('/api', file_handler)
            ('/', file_handler)
        ])

Here is some sample output from curl:

    prompt$ curl -s 'http://192.168.1.180/api/stats' | python -m json.tool
    {
        "esp": {
            "flash_id": 1327328,
            "flash_size": 1048576,
            "free_mem": 8456
        },
        "gc": {
            "mem_alloc": 29024,
            "mem_free": 7264
        },
        "machine": {
            "freq": 80000000,
            "unique_id": "0xBBB81500"
        },
        "network": {
            "ap": {
                "ifconfig": {
                    "dns": "192.168.1.1",
                    "gateway": "192.168.4.1",
                    "ip": "192.168.4.1",
                    "subnet": "255.255.255.0"
                },
                "status": "Unknown wlan status: -1"
            },
            "phy_mode": "MODE_11N",
            "sta": {
                "ifconfig": {
                    "dns": "192.168.1.1",
                    "gateway": "192.168.1.1",
                    "ip": "192.168.1.180",
                    "subnet": "255.255.255.0"
                },
                "status": "STAT_GOT_IP"
            }
        },
        "sys": {
            "byteorder": "little",
            "implementation": {
                "name": "micropython",
                "version": [
                    1,
                    8,
                    6
                ]
            },
            "maxsize": 2147483647,
            "modules": [
                "webrepl",
                "utcp_server",
                "ulog",
                "websocket_helper",
                "stats_api",
                "console_sink",
                "http_file_handler",
                "flashbdev",
                "http_api_handler",
                "uhttpd",
                "webrepl_cfg"
            ],
            "path": [
                "",
                "/lib",
                "/"
            ],
            "platform": "esp8266",
            "version": "3.4.0"
        }
    }



## TODO

Document the handler mechanism and techniques for implementing REST-ful APIs.