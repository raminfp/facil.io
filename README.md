# facil.io - the C WebApp mini-framework

**Notice: The *master* branch is the development branch. Please select the *stable* branch for the latest release or select a release version.**

[facil.io](http://facil.io) is a C mini-framework for web applications and includes a fast HTTP and Websocket server, as well as support for custom protocols.

[facil.io](http://facil.io) powers the [HTTP/Websockets Ruby Iodine server](https://github.com/boazsegev/iodine) and it can easily power your application as well.

[facil.io](http://facil.io) provides high performance TCP/IP network services to Linux / BSD (and macOS) by using an evented design and provides an easy solution to [the C10K problem](http://www.kegel.com/c10k.html).

You can read more about [facil.io](http://facil.io) on the [facil.io](http://facil.io) website.

## Starting a new project with `facil.io`

To start a new project using the `facil.io` framework, run the following command in the terminal (change `appname` to whatever you want):

     $ bash <(curl -s https://raw.githubusercontent.com/boazsegev/facil.io/master/scripts/new) appname

This will create a new folder called `appname` (or whatever you decide), downloads a copy of the stable branch, move the HTTP boiler plate code to the `appname/dev` folder and run `make clean` (which is required to build the `tmp` folder structure).

Once the boiler-plate code is ready, edit the `makefile` to remove any generic features you don't need, such as the `DUMP_LIB` feature, the `DEBUG` flag or the `DISAMS` disassembler.

Next, edit the boiler plate code to match your needs and start development.

*notice: boiler-plate code will only be available once version 0.5.2 is released.*

Credit to @benjcal for suggesting the script.

## Adding facil.io to an existing project

[facil.io](http://facil.io) is a source code library, so it's possible to simply copy the source code into an existing project and start using the library right away.

[facil.io](http://facil.io) also supports both `git` and CMake submodules.

To use the library in an existing project, clone the `git` repo and run:

     $ make libdump

This will dump all the library files into a folder called `libdump`.

The header files are in `libdump/include` and the source files are in `libdump/src`. The folder `libdump/all` contains all the source and header files mixed together.

Copy these files to your project, as required by your project's folder structure and start using the library.

## Three Quick Examples

Writing HTTP and Websocket services in C is easy with [facil.io](http://facil.io).

### HTTP/1.1

The simplest example, of course, would be the famous "Hello World" application... this is so easy, it's practically boring (so we add custom headers and cookies):

```c
#include "http.h"

void on_request(http_request_s* request) {
  http_response_s * response = http_response_create(request);
  // http_response_log_start(response); // logging ?
  http_response_set_cookie(response, .name = "my_cookie", .value = "data");
  http_response_write_header(response, .name = "X-Data", .value = "my data");
  http_response_write_body(response, "Hello World!\r\n", 14);
  http_response_finish(response);
}

int main() {
  char* public_folder = NULL;
  // listen on port 3000, any available network binding (NULL == 0.0.0.0)
  http_listen("3000", NULL, .on_request = on_request,
               .public_folder = public_folder, .log_static = 0);
  // start the server
  facil_run(.threads = 16);
}
```

### Websockets

[facil.io](http://facil.io) really shines when it comes to Websockets and real-time applications, where the `kqueue`/`epoll` engine gives the framework a high performance running start.

Here's a full-fledge example of a Websocket echo server, a Websocket broadcast server and an HTTP "Hello World" (with an optional static file service) all rolled into one:

```c
#include "websockets.h" // includes the "http.h" header

#include <stdio.h>
#include <stdlib.h>

/* ******************************
The Websocket echo implementation
*/

void ws_open(ws_s * ws) {
  fprintf(stderr, "Opened a new websocket connection (%p)\n", (void * )ws);
}

void ws_echo(ws_s * ws, char * data, size_t size, uint8_t is_text) {
  // echos the data to the current websocket
  websocket_write(ws, data, size, is_text);
}

void ws_shutdown(ws_s * ws) { websocket_write(ws, "Shutting Down", 13, 1); }

void ws_close(ws_s * ws) {
  fprintf(stderr, "Closed websocket connection (%p)\n", (void * )ws);
}

/* ********************
The HTTP implementation
*/

void on_request(http_request_s * request) {
  http_response_s * response = http_response_create(request);
  http_response_log_start(response); // logging
  // websocket upgrade.
  if (request->upgrade) {
    websocket_upgrade(.request = request, .on_message = ws_echo,
                      .on_open = ws_open, .on_close = ws_close, .timeout = 40,
                      .on_shutdown = ws_shutdown, .response = response);
    return;
  }
  // HTTP response
  http_response_write_body(response, "Hello World!", 12);
  http_response_finish(response);
}

/****************
The main function
*/

#define THREAD_COUNT 1
int main(void) {
  const char* public_folder = NULL;
  http_listen("3000", NULL, .on_request = on_request,
               .public_folder = public_folder, .log_static = 1);
  facil_run(.threads = THREAD_COUNT);
  return 0;
}
```

### A Custom Protocol (an Echo example)

[facil.io](http://facil.io)'s API is designed for both simplicity and an object oriented approach, using network protocol objects and structs to avoid bloating function arguments and to provide sensible default behavior.

Here's a simple Echo example (test with telnet to port `"3000"`).

```c
#include "facil.h" // the core library header

// Performed whenever there's pending incoming data on the socket
static void perform_echo(intptr_t uuid, protocol_s * prt) {
  (void)prt;
  char buffer[1024] = {'E', 'c', 'h', 'o', ':', ' '};
  ssize_t len;
  while ((len = sock_read(uuid, buffer + 6, 1018)) > 0) {
    sock_write(uuid, buffer, len + 6);
    if ((buffer[6] | 32) == 'b' && (buffer[7] | 32) == 'y' &&
        (buffer[8] | 32) == 'e') {
      sock_write(uuid, "Goodbye.\n", 9);
      sock_close(uuid); // closes after `write` had completed.
      return;
    }
  }
}
// performed whenever "timeout" is reached.
static void echo_ping(intptr_t uuid, protocol_s * prt) {
  (void)prt;
  sock_write(uuid, "Server: Are you there?\n", 23);
}
// performed during server shutdown, before closing the socket.
static void echo_on_shutdown(intptr_t uuid, protocol_s *prt) {
  (void)prt;
  sock_write(uuid, "Echo server shutting down\nGoodbye.\n", 35);
}
// performed after the socket was closed and the currently running task had
// completed.
static void destroy_echo_protocol(protocol_s * echo_proto) {
  if (echo_proto) // always error check, even if it isn't needed.
    free(echo_proto);
  fprintf(stderr, "Freed Echo protocol at %p\n", (void * )echo_proto);
}
// performed whenever a new connection is accepted.
static inline protocol_s *create_echo_protocol(intptr_t uuid, void *arg) {
  // create a protocol object
  protocol_s *echo_proto = malloc(sizeof( * echo_proto));
  // set the callbacks
  * echo_proto = (protocol_s){
      .service = "echo",
      .on_data = perform_echo,
      .on_shutdown = echo_on_shutdown,
      .ping = echo_ping,
      .on_close = destroy_echo_protocol,
  };
  // write data to the socket and set timeout
  sock_write(uuid, "Echo Service: Welcome. Say \"bye\" to disconnect.\n", 48);
  facil_set_timeout(uuid, 10);
  // print log
  fprintf(stderr, "New Echo connection %p for socket UUID %p\n",
          (void * )echo_proto, (void * )uuid);
  // return the protocol object to attach it to the socket.
  return echo_proto;
  (void)arg; // we don't use this
}
// creates and runs the server
int main(void) {
  // listens on port 3000 for echo services.
  facil_listen(.port = "3000", .on_open = create_echo_protocol);
  // starts and runs the server
  facil_run(.threads = 10);
  return 0;
}
```

---

## Forking, Contributing and all that Jazz

Sure, why not. If you can add Solaris or Windows support to `libreact`, that could mean `lib-server` would become available for use on these platforms as well (as well as the HTTP protocol implementation and all the niceties).

If you encounter any issues, open an issue (or, even better, a pull request with a fix) - that would be great :-)

Hit me up if you want to:

* Help me write HPACK / HTTP2 protocol support.

* Help me design / write a generic HTTP routing helper library for the `http_request_s` struct.

* If you want to help me write a new SSL/TLS library or have an SSL/TLS solution we can fit into `lib-server` (as source code).

* If you want to help promote the library, that would be great as well. Perhaps publish [benchmarks](https://github.com/TechEmpower/FrameworkBenchmarks)) or share your story.

* Writing documentation into the `facil.io` website would be great. I keep the source code documentation fairly updated, but sometimes I can be a lazy bastard.
