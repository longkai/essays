Web Servers
===
## Web Servers Come in All Shapes and Sizes
A Web server processes HTTP request and servers responses. The term "web server" can refer either to web server software or to the particular device or computer dedicated to serving the web pages.

## Web Server Implementations
Web Servers implement HTTP and the related TCP connection handling.

## Set up connection -- accept a client connection, or close if the client is unwanted.

### Handleing New Connections

### Client Hostname Identification
Most Web servers can be configured to convert client IP address into client hostnames, using "reverse DNS". Since it slow down web transactions, ususaly disabled.

### Determing the Client User Through ident
for security reason, it was deprecated.

## Receive request -- read an HTTP request message for the internet.

### Parsing
When parsing the request message, the web server:
1. Parse the request line looking for the request method, the specified resource identifier(URI), and he version number, each separated by a single space, and ending with a carriage-return line-feed(CRLF) sequence.
2. Reads the message headers, each ending in CRLF
3. Detects the end-of-headers bland line, ending in CRLF(if present)
4. Read the request body, if any(length specified by the Content-Length header)

### Internal Representations of Messages
Internal data structure that makes message easy to manipulate.

### Connection Input/Output Processing Architectures
1. Single-thread web servers
2. Multiprocess and Multithreaded web server
3. Multiplexed I/O servers
4. Multiplexed Multithreaded web servers

## Process request -- interpret the request message and take action.
TODO

## Access resource -- access the resource specified in the message.

### Doctoots
Like the File system

### virtually hosted doctoors
Vitually hosted web servers host multiple web sites on the same web server, giving each site its own distinct document root on the server.

The server can tell the web sites using the HTTP Host header, of from distinct IP addresses.

### User home directory doctoors
``GET /~bob/index.html HTTP/1.1``

### Directory Listing
A web servers can receive request for directory URLs, where the path resolves to a directory, not a file. There are three different action would be taken:
1. Return an error
2. Return a special, default, "index file" instead of the directory.
3. Scan the directory, and return an HTML page containing the contents.

### Dynamic Content Resource Mapping
cgi-bin, java servlet, asp, etc.

### Server-Side Includes(SSI)

### Access Controls
refer HTTP authentication

## Send response -- send the response back to the client
For persistent Connections, the connection may stay open, in which case the server needs to be extra cautious to compute the Content-Length header correctly, or the client will have no way of knowing when a response ends.

## Construct response -- create the HTTP response message with the right headers.

### Response Entities(if there was a body)
1. A **Content-Type** header, describing the MIME type of the response body
2. A **Content-Length** header, describing the size of the response body
3. The actual message body content

### MIME Typing
mime.types, Magic typing, Explicit typing, Type negotiation. Web servers also can be configured to associate particular files with MIME types.

### Redirection
Permanently moved resources, Temporarily moved resources, URL augmentation, Load balancing, Server affinity, Canonicalizing directory names

## Log transaction -- place notes about the completed transaction in a log file.

### EOF
```yaml
background: /assets/images/default.jpg
date: 2015-09-26T12:31:18+08:00
hide: false
license: cc-40-by
location: Shenzhen
summary: 'HTTP: The Definitive Guide'
tags:
- Web
weather: ""
```
