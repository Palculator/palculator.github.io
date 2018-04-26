---
layout: post
title: Python & MAME IPC via Lua & sockets
categories: dodonbotchi python lua mame
date: 2018-04-26 22
---

An essential part of [my current project][1] will be inter-process
communication between Python and MAME, mainly because I wouldn't want to write
a project as elaborate as this in Lua, the only scripting language MAME
supports. Since the information on how to achieve that wasn't actually easy to
find, I thought I'd write it down here.

## Lua & sockets in MAME

The main way I've ever seen sockets being done in Lua was simply by using the,
apparently popular, [luasocket][2] library. MAME itself is not bundled with
that library, and, since it relies on a component written in C, not simply
added via standard Lua integration. Instead, the file I/O class MAME exposes to
Lua offers a way to open sockets, which is [documented here][3]. A socket is
opened by opening a "file" of the name `socket.<host>:<port>`. A mistake I made
based on information I found in [this post][4] was to create the respective
socket file with the flags `rwc`. A member of the `#mame-dev` IRC channel on
Freenode was friendly enough to point out that the `c`, as "create" would
imply, actually makes a server socket itself. There's no error for the port
being already in use by another socket, so I simply ended up being confused as
to why my client in MAME wasn't connecting to my Python host. The more you
know! Here's a short example of how to open a socket in MAME from within Lua:

{% highlight lua %}
local socket = emu.file('rw')
socket.open('socket.127.0.0.1:32512')
socket.write('HELLO')
socket.read(5)
{% endhighlight %}

## Python side

From Python, everything went as you'd expect: Open a server socket, bind it to
a certain port, wait for a client to connect, and then interact with the client
socket. Despite not being very special, here's an example of a Python server
that the above MAME Lua would connect to:

{% highlight python %}
import socket
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('127.0.0.1', 32512))
server.listen()
client, addr = server.accept()
print(str(client.recv(5), 'ascii'))
client.send(b'HELLO')
{% endhighlight %}

And that's it. In DoDonBotchi, this basic architecture will be used to get data
from MAME to Python and input from Python to MAME. Other projects that aren't
implemented in Lua can similarly expose MAME functionality over a socket. Have
fun!

[1]: {% post_url 2018-04-26-dodonbotchi %}
[2]: https://github.com/diegonehab/luasocket
[3]: https://github.com/mamedev/mame/blob/master/src/frontend/mame/luaengine.cpp#L814
[4]: http://www.mameworld.info/ubbthreads/showflat.php?Cat=&Number=370723&page=0&view=expanded&sb=5&o=&vc=1
