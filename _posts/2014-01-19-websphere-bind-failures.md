---
layout: post
title: "Understanding socket bind failures on WebSphere Application Server"
category: tech
tags:
 - WebSphere
 - TCP/IP
blogger: /2014/01/websphere-bind-failures.html
disqus: true
description: >
 Understand why WebSphere sometimes seems to have problems reopening TCP ports after a restart and what to do about it.
---

If you are familiar with WebSphere Application Server, then you probably have already noticed that sometimes WebSphere
seems to have problems reopening TCP ports after a restart. When this occurs, the following kind of error message is
logged repeatedly:

~~~
TCPC0003E: TCP Channel TCP_2 initialization failed. The socket bind failed for host * and port 9080. The port may already be in use.
~~~
{: class="wrap"}

Eventually WebSphere will succeed in binding to the port, as indicated by the following message:

~~~
TCPC0001I: TCP Channel TCP_2 is listening on host * (IPv6) port 9080.
~~~
{: class="wrap"}

IBM recently published a [technote][1] about this problem. It concludes by claiming that *there is no solution for this
limitation and [WebSphere] is working as designed*. That statement however misses some key points and is actually not
quite accurate.

First of all, it is important to understand why the issue occurs. The problem is that on some of the TCP sockets it
attempts to put into listen mode, WebSphere doesn't set the [`SO_REUSEADDR`][2] socket option. On Linux you can check
that using the tool I presented in a [previous blog post][3]. You will see that the `SO_REUSEADDR` option is set on the
sockets used by IIOP (`BOOTSTRAP_ADDRESS` and `ORB_LISTENER_ADDRESS`) and the SOAP JMX connector
(`SOAP_CONNECTOR_ADDRESS`), but not on the sockets used by the Web container (`WC_defaulthost`, `WC_defaulthost_secure`,
`WC_adminhost` and `WC_adminhost_secure`), the SIB service (`SIB_ENDPOINT_ADDRESS` and `SIB_ENDPOINT_SECURE_ADDRESS`)
and the core group service (`DCS_UNICAST_ADDRESS`).

The problem with not setting `SO_REUSEADDR` is that the bind to a port will fail as long as there are TCP connections in
state `TIME_WAIT` on that port. A connection will enter this state if the connection termination is initiated by the
local end. E.g. when WebSphere decides to close an idle HTTP connection, then that connection will end up in state
`TIME_WAIT` and prevent WebSphere from reopening the HTTP port after a restart. As can be seen in the
[TCP state diagram][4], a connection can only leave the `TIME_WAIT` state after the corresponding timeout is reached. At
this point the connection transitions to the `CLOSED` state and the system discards all information about the
connection. That is the reason why after several attempts, WebSphere eventually succeeds in binding to the port. It is
also the reason why IBM's technote suggests modifying the timeout value.

More background on the `TIME_WAIT` state and the `SO_REUSEADDR` option can be found in section 18.6,
"TCP State Transition Diagram" of W. Richard Stevens' classic textbook "TCP/IP Illustrated, Vol. 1":

>   The `TIME_WAIT` state is also called the 2MSL wait state. Every implementation must choose a value for the maximum
>   segment lifetime (MSL). It is the maximum amount of time any segment can exist in the network before being
>   discarded. [...]
>
>   Given the MSL value for an implementation, the rule is: when TCP performs an active close, and sends the final ACK,
>   that connection must stay in the `TIME_WAIT` state for twice the MSL. This lets TCP resend the final ACK in case
>   this ACK is lost (in which case the other end will time out and retransmit its final FIN).
>
>   Another effect of this 2MSL wait is that while the TCP connection is in the 2MSL wait, the socket pair defining that
>   connection (client IP address, client port number, server IP address, and server port number) cannot be reused. That
>   connection can only be reused when the 2MSL wait is over.
>
>   Unfortunately most implementations (i.e., the Berkeley-derived ones) impose a more stringent constraint. By default
>   a local port number cannot be reused while that port number is the local port number of a socket pair that is in the
>   2MSL wait. [...]
>
>   Some implementations and APIs provide a way to bypass this restriction. With the sockets API, the `SO_REUSEADDR`
>   socket option can be specified. It lets the caller assign itself a local port number that's in the 2MSL wait, but
>   we'll see that the rules of TCP still prevent this port number from being part of a connection that is in the 2MSL
>   wait.

The question is now why WebSphere doesn't use `SO_REUSEADDR`. The technote suggest that this is "by design", but fails
to explain the rationale behind that "design". In addition, the quote from Stevens' book clearly shows that setting
`SO_REUSEADDR` does no harm, and it is what most other server processes do.

There is another interesting aspect. I mentioned earlier that the problem only affects some ports opened by WebSphere,
in particular the ones used by the Web container, the SIB service and the core group services. Interestingly, these
ports are all managed by the [WebSphere channel framework][5]. They can easily be identified by looking at the "Ports"
page for the server in the admin console:

![Ports page in the admin console](screenshot.png)

Ports that have a link "View associated transports" are managed by the channel framework and don't use `SO_REUSEADDR`
(Note that this suggests that the problem also exists for the ports used by the SIP service not considered earlier).

It turns out that the channel framework actually supports a custom property [`soReuseAddr`][6] that can be used to
specify the value of the `SO_REUSEADDR` option. The corresponding documentation in the infocenter is interesting because
it explicitly presents that custom property as a solution for the bind problem, contradicting the statement made in the
technote that *there is no solution*:

>   Use the `soReuseAddr` custom property to control bind behavior. When the WebSphere Application Server is restarted,
>   if the inbound TCP channels have problems trying to bind the listening socket, errors are printed into the
>   `SystemOut` file until either the bind is successful or the number of allowed bind attempts has been passed. This
>   custom property helps to avoid repeated error messages during the bind process.
>
>   For inbound TCP channel binding environments, you can avoid the repeated error messages by using the `soReuseAddr`
>   custom property to affect TCP inbound channel processing. When `soReuseAddr` is set to 1, the TCP channel is forced
>   to try each bind attempt with the re-use option set to true on the socket. The restart of the WebSphere Application
>   Server processes first binding attempt, despite those sockets in `TIME_WAIT` state.

By setting the `soReuseAddr` property to 1 on all TCP inbound channels, it is indeed possible to avoid the TCPC0003E
error entirely. To configure that property on a given port using the admin console, start from the corresponding
"View associated transports" link and look for "TCP inbound channel". If you prefer to use admin scripting, write a
script that locates all configuration objects of type `TCPInboundChannel` and adds the `soReuseAddr` property to these
objects.

This is still not the complete story though. IBM's technote makes the following statement about the environments in
which the problem occurs:

>   This problem may occur on Red Hat Enterprise Linux Server Release 5.9 with WebSphere Application Server versions
>   6.1.0.45 - 8.5.0.2.

Obviously the problem is not specific to a particular Linux distribution or version, but that is not the point here.
What is more interesting is the range of WebSphere versions. The technote doesn't make it clear whether the problem only
exists in WAS versions up to 8.5.0.2 or whether 8.5.0.2 was simply the current WAS version when the technote was
written. It turns out that it is actually the former: in WAS 8.5.5.1 the behavior is indeed no longer the same. Now all
listening sockets are created with the `SO_REUSEADDR` option set by default, and the problem no longer exists.

This would suggest that IBM simply changed the default value of the `soReuseAddr` custom property to 1. However, a
simple test with `soReuseAddr=0` shows that WebSphere 8.5.5.1 actually completely ignores the property and always
enables `SO_REUSEADDR`, although the current version of the corresponding infocenter page still mentions the property
and specifies that its default value is 0.

[1]: http://www-01.ibm.com/support/docview.wss?uid=swg21645658
[2]: http://man7.org/linux/man-pages/man7/socket.7.html
[3]: /2013/12/19/inspecting-socket-options-on-linux.html
[4]: http://commons.wikimedia.org/wiki/File:Tcp_state_diagram_fixed.svg
[5]: http://www-01.ibm.com/support/docview.wss?uid=swg27037778
[6]: http://pic.dhe.ibm.com/infocenter/wasinfo/v8r5/topic/com.ibm.websphere.base.doc/ae/rrun_chain_tcpcustom.html
