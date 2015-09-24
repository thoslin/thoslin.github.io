---
layout: post
title: "Build a simple protocol over TCP"
date: 2015-09-24 08:13
#comments: true
categories:
---
Disclaimer: I am not an expert of TCP or designing protocols, this post is just about my learning experience of building a protocols over TCP :)

#### A rookie mistake

When I was playing with sockets. A rookie mistake I made is assuming that each message send implies a message receive, like the following example:

server.py

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.bind((socket.gethostname(), 2333))
sock.listen(1)
connection, address = sock.accept()
while True:
    data = connection.recv(1024)
    if not data:
        break
    print "Received: %s" % data
```

client.py

```python
import socket

socket_address = socket.gethostname(), 2333
connection = socket.create_connection(socket_address)
connection.send("Hello there!")
connection.send("Bye bye!")
connection.close()
```

Run the client. I got the following output:
```
Receive: Hello there!Bye bye!
```.

So two sends result in one receive, not two receives as expected. Hah. This is a misunderstanding of how TCP works.

TCP is a stream oriented protocol, not a packet/message oriented protocol like UDP. I'd like to use this analogy: TCP is like making a phone call, a connection must be established before both end is able to talk, and when you talk, data stream flows on the connection. While UDP is like you're sending a text message.

#### The boundary

However this rookie mistake got me thinking, when we're building an application on top of TCP socket, for example, a chatting application, how do we know where each message ends since they are a stream of data? Where's the boundary of two messages? There must be something up on the application level.

#### 1. Delimiter

Back to the phone call analogy, let's say foo is reading a poem to bar over the phone, how does bar know when foo finishes a line? how does bar know if foo finishes the whole poem? Does the wired connection do that for you? NO. But what we know from common sense is that, there's a pause when you finish a line, and maybe a longer pause when you finish the poem. Similarly, maybe we can put a pause in the end of each message? Just like [\r\n](http://www.w3.org/Protocols/rfc2616/rfc2616-sec2.html#sec2.2) in HTTP headers.

Here is an improved version of the previous code using \r\n as the delimiter:

server.py

```python

import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.bind((socket.gethostname(), 2333))
sock.listen(1)
connection, address = sock.accept()

while True:
    data = connection.recv(1024)
    if not data:
        break
    else:
        for line in data.split("\r\n"):
            if line:
                print "Received: %s" % line
```

client.py

```python
import socket

socket_address = socket.gethostname(), 2333
connection = socket.create_connection(socket_address)
connection.send("Hello there!\r\n")
connection.send("Bye bye!\r\n")
connection.close()
```

Now we got the separate output:
```
Received: Hello there!
Received: Bye bye!
```

The downside of this approach is that, when dealing with a message that is longer than 1024, you just get part of the message. We might need a buffer to receive message until we get a delimiter.

#### 2. Fix length or Prefix length

What if messages are all in fix length? Short message can be filled with empty string, something like:
```python
connection.send("Hello there!".ljust(140))
```

So server just need to keep reading fix length of bytes from socket. This works. However there is still a hard limit on the length of the message.

What if we tell the server the length of each message beforehand? We can do that by prefixing the message with the length of it. Yes! Just like the "Content-Length" header in HTTP.
```python
make_message = lambda x: str(len(x)).ljust(4) + x
connection.send(make_message("Hello there!"))
connection.send(make_message("Bye bye!"))
```

Here we prefix each message 4 bytes string indicating the length of the message. And server will first read the 4 bytes to get the length, then read as much bytes as that. The recvall function is to get the certain length of data, otherwise with simply recv, there's a chance we get just part of the transmitted data. Although in local machine the chance is low.

``` python
connection, address = sock.accept()

def recvall(conn, remains):
    buf = ""
    while remains:
        data = conn.recv(remains)
        if not data:
            break
        buf += data
        remains -= len(data)
    return buf

while True:
    data = recvall(connection, 4)
    if not data:
        break
    length = int(data)
    message = recvall(connection, length)

    print "Received: %s" % message
```

At this point, we have something like a protocol over the TCP layer, which is able to achieve the original goal.

#### Native protocol of Cassandra

Now that we have a protocol of our own, although simple and naive, I'd like to take a look at some serious protocol that built on TCP. Since I've been working with Cassandra a lot lately. I might as well just check their protocol.

[CQL](https://git-wip-us.apache.org/repos/asf?p=cassandra.git;a=blob_plain;f=doc/native_protocol.spec;hb=refs/heads/cassandra-1.2) is the protocol of Cassandra, which is built on TCP:

<quote>
  The CQL binary protocol is a frame based protocol. Frames are defined as:

      0         8        16        24        32
      +---------+---------+---------+---------+
      | version |  flags  | stream  | opcode  |
      +---------+---------+---------+---------+
      |                length                 |
      +---------+---------+---------+---------+
      |                                       |
      .            ...  body ...              .
      .                                       .
      .                                       .
      +----------------------------------------

</quote>

Frames can be regarded as what we called messages in previous examples. Except the first 32 bits, the length and body part is just what we used. So our approach looks practical.

So that's it, there must be more technical details regarding building a full-fledged protocol, but some fundamental things should work the same.
