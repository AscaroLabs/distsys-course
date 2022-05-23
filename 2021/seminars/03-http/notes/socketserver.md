# Socketserver

[Docs](https://docs.python.org/3/library/socketserver.html)

The `socketserver` module simplifies the task of writing network servers.

There are four basic concrete server classes:
- class socketserver.**TCPServer**(server_address, RequestHandlerClass, bind_and_activate=True)
- class socketserver.**UDPServer**(server_address, RequestHandlerClass, bind_and_activate=True)
- class socketserver.**UnixStreamServer**(server_address, RequestHandlerClass, bind_and_activate=True)
- class socketserver.**UnixDatagramServer**(server_address, RequestHandlerClass, bind_and_activate=True)

These four classes process requests _synchronously_, each request must be completed before the next request can be started.

Creating a server requires several steps. First, you must create a request handler class by subclassing the `BaseRequestHandler` class and overriding its `handle()` method; this method will process incoming requests.  
Second, you must instantiate one of the server classes, passing it the server’s address and the request handler class. It is recommended to use the server in a `with` statement.  
Then call the `handle_request()` or `serve_forever()` method of the server object to process one or many requests.
Finally, call `server_close()` to close the socket (unless you used a `with` statement). 

When inheriting from `ThreadingMixIn` for threaded connection behavior, you should explicitly declare how you want your threads to behave on an abrupt shutdown. The `ThreadingMixIn` class defines an attribute `daemon_threads`, which indicates whether or not the server should wait for thread termination. You should set the flag `explicitly` if you would like threads to behave autonomously; the default is `False`, meaning that Python will not exit until all threads created by `ThreadingMixIn` have exited.

## Server Objects

_class_ socketserver.**BaseServer**(server_address, RequestHandlerClass)  
This is the superclass of all Server objects in the module. It defines the interface, given below, but does not implement most of the methods, which is done in subclasses. The two parameters are stored in the respective `server_address` and `RequestHandlerClass` attributes.

- **`fileno()`** return an integer file descriptor for the socket on which the server is listening. 
- **`handle_request()`** process a single request; This function calls the following methods in order:
  - get_request()
  - verify_request()
  - process_request()
  - handle_error() if handle raises an exemption
  -  If no request is received within timeout seconds, handle_timeout() will be called and handle_request() will return
- **`serve_forever(poll_interval=0.5)`**
- **`shutdown()`**
- **`address_family`** The family of protocols to which the server’s socket belongs. Common examples are `socket.AF_INET` and `socket.AF_UNIX`
- **`RequestHandlerClass`** The user-provided request handler class; an instance of this class is created for each request.
- **`server_address`** The address on which the server is listening. The format of addresses varies depending on the protocol family;For internet protocols, this is a tuple containing a string giving the address, and an integer port number: _('127.0.0.1', 80)_, for example.
- **`socket`** The socket object on which the server will listen for incoming requests.

## Request Handler Objects

_class_ socketserver.**BaseRequestHandler**  

This is the superclass of all request handler objects. It defines the interface, given below. A concrete request handler subclass must define a new `handle()` method, and can override any of the other methods. A new instance of the subclass is created for each request.
- **`setup()`** Called before the `handle()` method to perform any initialization actions required. The default implementation does nothing.
- **`handle()`** This function must do all the work required to service a request. The default implementation does nothing. Several instance attributes are available to it; the request is available as _`self.request`_; the client address as _`self.client_address`_; and the server instance as _`self.server`_, in case it needs access to per-server information.
- **`finish()`** Called after the `handle()` method to perform any clean-up actions required. The default implementation does nothing. If `setup()` raises an exception, this function will not be called.

_class_ socketserver.**StreamRequestHandler**  
_class_ socketserver.**DatagramRequestHandler**  
These `BaseRequestHandler` subclasses override the `setup()` and `finish()` methods, and provide _`self.rfile`_ and _`self.wfile`_ attributes. The _`self.rfile`_ and _`self.wfile`_ attributes can be read or written, respectively, to get the request data or return data to the client.

## Examples

### __socketserver.TCPServer__ Example

This is the server side:
```python
import socketserver

class MyTCPHandler(socketserver.BaseRequestHandler):
    """
    The request handler class for our server.

    It is instantiated once per connection to the server, and must
    override the handle() method to implement communication to the
    client.
    """

    def handle(self):
        # self.request is the TCP socket connected to the client
        self.data = self.request.recv(1024).strip()
        print("{} wrote:".format(self.client_address[0]))
        print(self.data)
        # just send back the same data, but upper-cased
        self.request.sendall(self.data.upper())

if __name__ == "__main__":
    HOST, PORT = "localhost", 9999

    # Create the server, binding to localhost on port 9999
    with socketserver.TCPServer((HOST, PORT), MyTCPHandler) as server:
        # Activate the server; this will keep running until you
        # interrupt the program with Ctrl-C
        server.serve_forever()
```
An alternative request handler class that makes use of streams (file-like objects that simplify communication by providing the standard file interface):
```python
class MyTCPHandler(socketserver.StreamRequestHandler):

    def handle(self):
        # self.rfile is a file-like object created by the handler;
        # we can now use e.g. readline() instead of raw recv() calls
        self.data = self.rfile.readline().strip()
        print("{} wrote:".format(self.client_address[0]))
        print(self.data)
        # Likewise, self.wfile is a file-like object used to write back
        # to the client
        self.wfile.write(self.data.upper())
```
Client side:
```python
import socket
import sys

HOST, PORT = "localhost", 9999
data = " ".join(sys.argv[1:])

# Create a socket (SOCK_STREAM means a TCP socket)
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
    # Connect to server and send data
    sock.connect((HOST, PORT))
    sock.sendall(bytes(data + "\n", "utf-8"))

    # Receive data from the server and shut down
    received = str(sock.recv(1024), "utf-8")

print("Sent:     {}".format(data))
print("Received: {}".format(received))
```
The output of the example should look something like this:
```sh
$ python TCPServer.py
127.0.0.1 wrote:
b'hello world with TCP'
127.0.0.1 wrote:
b'python is nice'
```
```sh
$ python TCPClient.py hello world with TCP
Sent:     hello world with TCP
Received: HELLO WORLD WITH TCP
$ python TCPClient.py python is nice
Sent:     python is nice
Received: PYTHON IS NICE
```
