---
lang: en
layout: doc
permalink: /doc/qrexec-internals/
redirect_from:
- /doc/qrexec3-implementation/
- /en/doc/qrexec3-implementation/
- /doc/Qrexec3Implementation/
- /wiki/Qrexec3Implementation/
ref: 39
title: 'Qrexec: Qubes RPC internals'
---

(*This page details the current implementation of qrexec (qrexec3).
A [general introduction](/doc/qrexec/) to qrexec is also available.*)

The qrexec framework consists of a number of processes communicating with each other using a common IPC protocol, described in detail below.

Components residing in the same domain (`qrexec-client-vm` to `qrexec-agent`, `qrexec-client` to `qrexec-daemon`) use local sockets as the underlying transport medium.
Components in separate domains (`qrexec-daemon` to `qrexec-agent`, data channel between `qrexec-agent`s) use vchan links.
Because of [vchan limitation](https://github.com/qubesos/qubes-issues/issues/951), it is not possible to establish qrexec connection back to the source domain.

## Dom0 tools implementation

The following programs handle parts of the framework: sending and receiving requests, verifying permissions, and administering connections.
These tools are not designed to be used by users directly.

### qrexec-daemon

`/usr/sbin/qrexec-daemon`

One instance is required for every active domain.
`qrexec-daemon` is responsible for both:

- handling execution and service requests from **dom0** (source: `qrexec-client` and other `qrexec-daemon` instances); and
- handling service requests from the associated domain (source: `qrexec-client-vm` via `qrexec-agent`).

Command line usage:

`qrexec-daemon domain-id domain-name [default user]`

- `domain-id`: Numeric Qubes ID assigned to the associated domain.
- `domain-name`: Associated domain name.
- `default user`: Optional. If passed, `qrexec-daemon` uses this user as default for all execution requests that don't specify one.

### qrexec-client

`/usr/bin/qrexec-client`

Used to pass execution and service requests to `qrexec-daemon`.
Starting with v4.2.19, `qrexec-daemon` does this itself, so `qrexec-client` is only used for requests made by dom0 and for unit tests.

Command line usage:

- `-d target-domain-name`: Specifies the target for the execution/service request.
- `-l local-program`: Optional. If present, `local-program` is executed and its stdout/stdin are used when sending/receiving data to/from the remote peer.
  If `local-program` starts with `QUBESRPC`, it is interpreted as a a service request and executed in dom0.
- `-e`: Optional. If present, stdout/stdin are not connected to the remote peer. Only process creation status code is received.
- `-c <request-id,src-domain-name,src-domain-id>`: used for connecting a VM-VM service request by `qrexec-policy` or `qrexec-daemon`.
  Details described below in the service example.
  Newer versions of `qrexec-daemon` and `qrexec-policy` no longer use this, but support remains in `qrexec-client`.
- `cmdline`: Command line to pass to `qrexec-daemon` as the execution/service request. Service request format is described below in the service example.
- `-k`: Optional. If present, the destination VM is killed (via Admin API call) after the call ends.  It should also be killed if an error occurs, but this
  does not always happen due to a bug.

In R4.1 and below, `qrexec-client` is invoked by the qrexec policy engine for every VM -> VM service call.
In early R4.2 releases (`qubes-core-qrexec-dom0` below 4.2.19), it is invoked by `qrexec-daemon` for such calls.
In later R4.2 releases (`qubes-core-qrexec-dom0` 4.2.19+), and in R4.3+, it is only used for calls made by dom0 and for unit tests.

## VM tools implementation

### qrexec-agent

`/usr/lib/qubes/qrexec-agent`

One instance runs in each active domain.
Responsible for:

- Handling service requests from `qrexec-client-vm` and passing them to connected `qrexec-daemon` in dom0.
- Executing associated `qrexec-daemon` execution/service requests.

The `qrexec-agent` command takes no parameters.

### qrexec-client-vm

`/usr/bin/qrexec-client-vm`

Runs in an active domain.
Used to pass service requests to `qrexec-agent`.

Command line usage:

`qrexec-client-vm target-domain-name service-name local-program [local program arguments]`

- `target-domain-name`: Target domain for the service request. Source is the current domain.
- `service-name`: Requested service name.
- `local-program`: `local-program` is executed locally and its stdin/stdout are connected to the remote service endpoint.

## Qrexec protocol details

The qrexec protocol is message-based.
All messages share a common header followed by an optional data packet.

```c
/* uniform for all peers, data type depends on message type */
struct msg_header {
   uint32_t type;           /* message type */
   uint32_t len;            /* data length */
};
```

When two peers establish connection, the server sends `MSG_HELLO` followed by `peer_info` struct:

```c
struct peer_info {
   uint32_t version; /* qrexec protocol version */
};
```

The client then should reply with its own `MSG_HELLO` and `peer_info`.
The lower of two versions define protocol used for this connection.
If either side does not support this version, the connection is closed.

Details of all possible use cases and the messages involved are described below.

### dom0: request execution of `cmd` in domX

![qrexec internals diagram dom0-vm](/attachment/doc/qrexec-dom0-vm.png)

- **dom0**: `qrexec-client` is invoked in **dom0** as follows:

      qrexec-client -d domX [-l local_program] user:cmd

  (If `local_program` is set, `qrexec-client` executes it and uses that child's stdin/stdout in place of its own when exchanging data with `qrexec-agent` later.)

  `qrexec-client` translates that request into a `MSG_EXEC_CMDLINE` message sent to `qrexec-daemon`, with `connect_domain` set to 0 (connect to **dom0**) and `connect_port` also set to 0 (allocate a port).

- **dom0**: `qrexec-daemon` allocates a free port (in this case 513), and sends a `MSG_EXEC_CMDLINE` back to the client with connection parameters (**domX** and 513) and with command field empty.

  `qrexec-client` disconnects from the daemon, starts a vchan server on port 513 and awaits connection.

  Then, `qrexec-daemon` passes on the request as `MSG_EXEC_CMDLINE` message to the `qrexec-agent` running in **domX**. In this case, the connection parameters are **dom0** and 513.

- **domX**: `qrexec-agent` receives `MSG_EXEC_CMDLINE`, and starts the command (`user:cmd`, or `cmd` as user `user`). If possible, this is actually delegated to a separate server (`qrexec-fork-server`) also running on domX.

  After starting the command, `qrexec-fork-server` connects to `qrexec-client` in **dom0** over the provided vchan port 513.

- Data is forwarded between the `qrexec-client` in **dom0** and the command executed in **domX** using `MSG_DATA_STDIN`, `MSG_DATA_STDOUT` and `MSG_DATA_STDERR`.

  Empty messages (with data `len` field set to 0 in `msg_header`) are an EOF marker. Peer receiving such message should close the associated input/output pipe.

  When `cmd` terminates, **domX**'s `qrexec-fork-server` sends `MSG_DATA_EXIT_CODE` header to `qrexec-client` followed by the exit code (**int**).

### domX: request execution of service `admin.Service` in dom0

![qrexec internals diagram vm-dom0](/attachment/doc/qrexec-vm-dom0.png)

- **domX**: `qrexec-client-vm` is invoked as follows:

      qrexec-client-vm dom0 admin.Service [local_program] [params]

  (If `local_program` is set, it will be executed in **domX** and connected to the remote command's stdin/stdout).

  `qrexec-client-vm` connects to `qrexec-agent` and requests service execution (`admin.Service`) in **dom0**.

  `qrexec-agent` assigns an internal identifier to the request. It's based on a file descriptor of the connected `qrexec-client-vm`: in this case, `SOCKET11`.

  `qrexec-agent` forwards the request (`MSG_TRIGGER_SERVICE3`) to its corresponding `qrexec-daemon` running in dom0.

- **dom0**: `qrexec-daemon` receives the request.
  It tries to connect to `qrexec-policy-daemon` if possible.
  If that is not possible, it triggers `qrexec-policy-exec` program.
  In both cases, it passes all necessary parameters: source domain **domX**, target domain **dom0**, service `admin.Service` and identifier `SOCKET11`.

  `qrexec-policy-daemon` or `qrexec-policy-exec` evaluates if the RPC should be allowed or denied, possibly also launching a GUI confirmation prompt.

  If the RPC is denied, `qrexec-daemon` sends a `MSG_SERVICE_REFUSED` back.

  The protocol used by `qrexec-policy-daemon` is documented in [qrexec-policy-daemon.rst](https://github.com/QubesOS/qubes-core-qrexec/blob/main/Documentation/qrexec-policy-daemon.rst).

- **dom0**: If the RPC is allowed, what happens depends on the Qubes OS version:

  - In R4.1 and below, `qrexec-policy-daemon` or `qrexec-policy-exec` will launch a `qrexec-client` with the right command:

        qrexec-client -d dom0 -c domX,X,SOCKET11 "QUBESRPC admin.Service+ domX name dom0"

  - In R4.2 before qrexec 4.2.19, `qrexec-daemon` will instead launch a `qrexec-client`, with the same command.

        qrexec-client -d dom0 -c domX,X,SOCKET11 "QUBESRPC admin.Service+ domX name dom0"

  - Starting with qrexec 4.2.19, `qrexec-daemon` executes the command directly.

  The `-c domX,X,SOCKET11` are parameters indicating how connect back to **domX** and pass its input/output.

  The command parameter describes the RPC call: it contains service name and argument (`admin.Service+`), source domain (`domX`) and target description (`name dom0`, could also be e.g. `keyword @dispvm`).
  The target description is important in case the original target wasn't dom0, but the service is executing in dom0.
  The detailed syntax of the command can be found in [the qrexec command specification](https://github.com/QubesOS/qubes-core-qrexec/blob/main/doc/execution.rst).

  `qrexec-client` (R4.2 and below) or `qrexec-daemon` (R4.3 and up) connects to the `qrexec-daemon` for **domX** and sends a `MSG_SERVICE_CONNECT` with connection parameters (**dom0**, and port 0, indicating a port should be allocated) and request identifier (`SOCKET11`).

  `qrexec-daemon` allocates a free port (513) and sends back connection parameters to `qrexec-client` or `qrexec-daemon` (**domX** port 513).

  `qrexec-client` or `qrexec-client` starts the command, and tries to connect to **domX** over the provided port 513.

  Then, `qrexec-daemon` forwards the connection request (`MSG_SERVICE_CONNECT`) to `qrexec-agent` running in **domX**, with the right parameters (**dom0** port 513, request `SOCKET11`).

- **dom0**: Because the command has the form `QUBESRPC: ...`, it is started through the `qubes-rpc-multiplexer` program with the provided parameters (`admin.Service domX name dom0`). That program finds and executes the necessary script in `/etc/qubes-rpc/`.
  Starting in R4.3, `qubes-rpc-multiplexer` is skipped and `qrexec-daemon` executes the command itself.

- **domX**: `qrexec-agent` receives the `MSG_SERVICE_CONNECT` and passes the connection parameters back to the connected `qrexec-client-vm`. It identifies the `qrexec-client-vm` by the request identifier (`SOCKET11` means file descriptor 11).

  `qrexec-client-vm` starts a vchan server on 513 and receives a connection from `qrexec-client`.

- Data is forwarded between **dom0** and **domX** as in the previous example (dom0-VM).

### domX: invoke execution of qubes service `qubes.Service` in domY

![qrexec internals diagram vm-vm](/attachment/doc/qrexec-vm-vm.png)

- **domX**: `qrexec-client-vm` is invoked as follows:

      qrexec-client-vm domY qubes.Service [local_program] [params]

  (If `local_program` is set, it will be executed in **domX** and connected to the remote command's stdin/stdout).

- The request is forwarded as `MSG_TRIGGER_SERVICE3` to `qrexec-daemon` running in **dom0**
  then to `qrexec-policy-daemon`.  If the call is allowed, `qrexec-daemon` then executes the
  command (>= 4.2.19) or spawns a `qrexec-client` to do that (< 4.2.19).

  This is the same as in the previous example (VM-dom0).

- **dom0**: If the RPC is allowed, `qrexec-daemon` will first check if **domY** needs to be started.
  If it does, it makes an `admin.vm.Start` call to start the VM.
  "VM already running" is treated as success.
  Other errors are treated as if the RPC was denied.

  `qrexec-daemon` for **domX** constructs a command:

      DEFAULT:QUBESRPC qubes.Service domX

  The command parameter describes the service call: it contains the username (or `DEFAULT`), service name (`qubes.Service`) and source domain (`domX`).

  `qrexec-daemon` for **domX** will then send a `MSG_EXEC_CMDLINE` message to `qrexec-daemon` for **domY**. The message will be with port number 0, requesting port allocation.

  `qrexec-daemon` for **domY** will allocate a port (513) and send it back. It will also send a `MSG_EXEC_CMDLINE` to its corresponding agent. (It will also translate `DEFAULT` to the configured default username).

  Then, `qrexec-daemon` for **domX** will also send `MSG_SERVICE_CONNECT` message to **domX**'s agent, indicating that it should connect to **domY** over port 513.

- **domX**: `qrexec-agent` receives a `MSG_SERVICE_CONNECT` with connection parameters (**domY** port 513) and request identifier (`SOCKET11`). It sends the connection parameters back to the right `qrexec-client-vm`.

  `qrexec-client-vm` starts a vchan server on port 513. Note that this is different than in the other examples: `MSG_SERVICE_CONNECT` means you should start a server, `MSG_EXEC_CMDLINE` means you should start a client.

- **domY**: `qrexec-agent` receives a `MSG_EXEC_CMDLINE` with the command to execute (`user:QUBESRPC...`) and connection parameters (**domX** port 513).

  Since this is not a socket-based service and is running as the default user, it forwards the request to `qrexec-fork-server`, which handles the command and connects to **domX** over the provided port.
  If this was a socket-based service or if it was running as a non-default user, it would handle the command itself.

  Because the command is of the form `QUBESRPC ...`, `qrexec-fork-server` starts it using `qubes-rpc-multiplexer` program, which finds and executes the necessary script in `/etc/qubes-rpc/`.
  In R4.3+, the command is found and executed directly from the `qrexec-fork-server` or `qrexec-agent` binary, without the `qubes-rpc-multiplexer` script.

- After that, the data is passed between **domX** and **domY** as in the previous examples (dom0-VM, VM-dom0).

## `qrexec-policy` implementation

`qrexec-policy` is a mechanism for evaluating whether an RPC call should be allowed. For introduction, see [Qubes RPC administration](/doc/qrexec/#qubes-rpc-administration).

### `qrexec-policy-daemon`

This is a service running in dom0. It is called by `qrexec-daemon` and is responsible for evaluating the request.

The daemon listens on a socket (`/var/run/qubes/policy.sock`). It accepts requests in the format described in [qrexec-policy-daemon.rst](https://github.com/QubesOS/qubes-core-qrexec/blob/master/doc/qrexec-policy-daemon.rst) and replies with `result=allow/deny`.

A standalone version is called `qrexec-policy-exec` and is available as a fallback.

### `qrexec-policy-agent`

This is a service running in the GuiVM. It is called by `qrexec-policy-daemon` in order to display prompts and notifications to the user.

It is a [socket-based Qubes RPC service](/doc/qrexec-socket-services/). Requests are in JSON format, and response is simple ASCII.

There are two endpoints:

- `policy.Ask` - ask the user about whether to execute a given action
- `policy.Notify` - notify the user about an action.

See [qrexec-policy-agent.rst](https://github.com/QubesOS/qubes-core-qrexec/blob/main/Documentation/qrexec-policy-agent.rst) for protocol details.
