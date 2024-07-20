---
lang: en
layout: doc
permalink: /doc/qrexec-socket-services/
ref: 42
title: 'Qrexec: socket-based services'
---

*This page describes how to implement and use new socket-backed services for qrexec. See [qrexec](/doc/qrexec/) for general overview of the qrexec framework.*

As of Qubes 4.1, qrexec allows implementing services not only as executable files, but also as Unix sockets.
This allows Qubes RPC requests to be handled by a server running in a VM and listening for connections.

Starting in R4.2 with qrexec 4.2.19 or later, it is also possible for a service to be implemented as a TCP socket.

## How it works

When a Qubes RPC service is invoked,
qrexec searches for a file that handles it in the qubes-rpc directories (`/etc/qubes-rpc` or `/usr/local/etc/qubes-rpc`).
If the file is a Unix socket, qrexec will try to connect to it.
If the file is a symbolic link to a path that starts with `/dev/tcp`, qrexec will parse the link to obtain an IP address and TCP port number:

- If the path is exactly `/dev/tcp`, the IP address and port number will be taken from the service argument.
  Everything after the last `+` in the service argument is considered to be the service argument.

Before passing user input, the socket service will receive a null-terminated service descriptor, i.e. the part after `QUBESRPC`.
When running in a VM, this is:

```
<service_name> <source>\0
```

When running in dom0, it is:

```
<service_name> <source> <target_type> <target>\0
```

(The target type can be `name`, in which case target is a domain name, or `keyword`, in which the target is a keyword like `@dispvm`).

To disable passing this descriptor, add `skip-service-descriptor=true` to the service configuration file.

Afterwards, data provided by the service's user (as stdin) is sent into the socket, and data received from the socket is sent back to the user (as stdout).
When the service closes the socket, or if EOF is received on both ends of the connection, an exit code of 0 is sent back to the user.
Adding `exit-on-service-eof=true` to the configuration file causes qrexec to exit when the service closes its connection for writing,
even if more data might be available from the service's user.
Similarly, `exit-on-client-eof=true` in the configuration file causes qrexec to exit when the user closes the write end of the connection,
even if the service might have more data to send.

`skip-service-descriptor=true`, `exit-on-service-eof=true`, and `exit-on-client-eof=true` are only valid for socket-based services.
If they are present for an executable service, the config option will be ignored and a warning will be logged.

### Differences from executable-based services

From the user point of view, the socket-based service behaves almost like an executable-based one.
Here are the differences:

* There is no stderr (the socket provides only one output stream).
  Before 4.2.20, stderr was not closed until the command exited.
  Starting with 4.2.20, stderr is closed immediately.
* There is no exit code.
  When the socket connection is closed, exit code 0 is sent to the user.
  However, if the socket cannot be connected to, exit code 125 will be sent.

## Recommended use

Create a program that binds to path *outside* `/etc/qubes-rpc`, such as `/var/run/my-daemon.sock`.
Put a symlink in `/etc/qubes-rpc`, e.g. `ln -s /var/run/my-daemon.sock /etc/qubes-rpc/qubes.Service`.

If your program handles multiple services, create multiple symlinks.
You can dispatch based on the service descriptor.

Do not run the program as root unless necessary.

You can use systemd and socket activation so that the program is started only when the service is invoked.
See the below example.

## Example: `qrexec-policy-agent`

`qrexec-policy-agent` is the program that handles "ask" prompts for Qubes RPC calls.
It is a good example of an application that:
* Uses Python and asyncio.
* Runs as a daemon, to save some overhead on starting process.
* Runs as a normal user.
  This is achieved using user's instance of systemd.
* Uses systemd socket activation.
  This way it can be installed in all VMs, but started only if it's ever needed.

See the [qubes-core-qrexec](https://github.com/QubesOS/qubes-core-qrexec/) repository for details.

### Systemd unit files

**/lib/systemd/user/qubes-qrexec-policy-agent.service**: This is the service configuration.

```
[Unit]
Description=Qubes remote exec policy agent
ConditionUser=!root
ConditionGroup=qubes
Requires=qubes-qrexec-policy-agent.socket

[Service]
Type=simple
ExecStart=/usr/bin/qrexec-policy-agent

[Install]
WantedBy=default.target
```

**/lib/systemd/user/qubes-qrexec-policy-agent.socket**: This is the socket file that will activate the service.

```
[Unit]
Description=Qubes remote exec policy agent socket
ConditionUser=!root
ConditionGroup=qubes
PartOf=qubes-qrexec-policy-agent.service

[Socket]
ListenStream=/var/run/qubes/policy-agent.sock

[Install]
WantedBy=sockets.target
```

Note the `ConditionUser` and `ConditionGroup` that ensure that the socket and service is started only as the right user

Start the socket using `systemctl --user start`.
Enable it using `systemctl --user enable`, so that it starts automatically.

```
systemctl --user start qubes-qrexec-policy-agent.socket
systemctl --user enable qubes-qrexec-policy-agent.socket
```

Alternatively, you can enable the service by creating a symlink:

```
sudo ln -s /lib/systemd/user/qubes-qrexec-policy-agent.socket /lib/systemd/user/sockets.target.wants/
```

### Link in qubes-rpc

`qrexec-policy-agent` will handle a Qubes RPC service called `policy.Ask`, so we add a link:

```
sudo ln -s /var/run/qubes/policy-agent.sock /etc/qubes-rpc/policy.Ask
```

### Python server with socket activation

Socket activation in systemd works by starting our program with the socket file already bound at a specific file descriptor.
It's a simple mechanism based on a few environment variables, but the canonical way is to use the `sd_listen_fds()` function from systemd library (or, in our case, its Python version).

Install the Python systemd library:

```
sudo dnf install python3-systemd
```

Here is the server code:

```python
import os
import asyncio
import socket

from systemd.daemon import listen_fds


class SocketService:
    def __init__(self, socket_path, socket_activated=False):
        self._socket_path = socket_path
        self._socket_activated = socket_activated

    async def run(self):
        server = await self.start()
        async with server:
            await server.serve_forever()

    async def start(self):
        if self._socket_activated:
            fds = listen_fds()
            if fds:
                assert len(fds) == 1, 'too many listen_fds: {}'.format(
                    listen_fds)
                sock = socket.socket(fileno=fds[0])
                return await asyncio.start_unix_server(self._client_connected,
                                                       sock=sock)

        if os.path.exists(self._socket_path):
            os.unlink(self._socket_path)
        return await asyncio.start_unix_server(self._client_connected,
                                               path=self._socket_path)

    async def _client_connected(self, reader, writer):
        try:
            data = await reader.read()
            assert b'\0' in data, data

            service_descriptor, data = data.split(b'\0', 1)

            response = await self.handle_request(service_descriptor, data)

            writer.write(response)
            await writer.drain()
        finally:
            writer.close()
            await writer.wait_closed()

    async def handle_request(self, service_descriptor, data):
        # process params, return response

        return response


def main():
    socket_path = '/var/run/qubes/policy-agent.sock'
    service = SocketService(socket_path)

    loop = asyncio.get_event_loop()
    loop.run_until_complete(service.run())


if __name__ == '__main__':
    main()
```

You can also use `qrexec/server.py` from [qubes-core-qrexec](https://github.com/QubesOS/qubes-core-qrexec/) repository, which is a variant of the above code - but note that currently it's somewhat more specific (JSON requests and ASCII responses; no target handling in service descriptors).

### Using the service

The service is invoked in the same way as a standard Qubes RPC service:

```
echo <input_data> | qrexec-client -d domX 'DEFAULT:QUBESRPC policy.Ask'
```

You can also connect to it locally, but remember to include the service descriptor:

```
echo -e 'policy.Ask dom0\0<input data>' | nc -U /etc/qubes-rpc/policy.Ask
```

## Further reading

* [Qrexec overview](/doc/qrexec/)
* [Qrexec internals](/doc/qrexec-internals/)
* [qubes-core-qrexec](https://github.com/QubesOS/qubes-core-qrexec/) repository - contains the above example
* [systemd.socket](https://www.freedesktop.org/software/systemd/man/systemd.socket.html) - socket unit configuration
* [Streams in Python asyncio](https://docs.python.org/3/library/asyncio-stream.html)
