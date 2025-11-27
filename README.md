# ZPR Demo

This repository contains support files for the containerized ZPR Demo as well as
configuration and scripts used to create new versions of the demo.

The rest of this file is about running the demo.  If you need to build a new release see
[README-DEV.md](README-DEV.md).

# Latest Demo Release

<!-- this is largely redundant with # How to run the demo -->

The latest release will be here in `main` and in a branch named `demo-YYYYMMDD`.

The demo consists of a container image, some binaries, and some files in this repo.

- The container image can be downloaded from GHCR in the [packages area](https://github.com/orgs/org-zpr/packages/container/package/zpr-demo%2Fzprdemo).
- The release binaries are in the [releases area](https://github.com/org-zpr/zpr-demo/releases).
- This git repo itself, [especially the configuration files in [release/conf](https://github.com/org-zpr/zpr-demo/tree/main/release/conf).


# Demo overview

## The pieces that will run

...
BAS (is:)

Visa Service

Node

...

## Network description

ZPR addresses are IPv6 addresses beginning with `fd5a:5052`. Whenever you connect by running an ***adapter*** with the `ph adapter` command, this creates a TUN virtual network interface and creates a routing table entry so that traffic for this prefix goes into the TUN. 

The ***substrate*** is the underlying IP network which "physically" connects an adapter to a ***node*** (or, one node to another node). An adapter has to be configured to know the node's IP address and port number on the substrate network (i.e., the ordinary network) so it can connect to it and tunnel traffic. This is called the node's ***docking port***.

In this demo, the Docker Compose file defines a Docker network called `substrate`, which each container (`node`, `bas`, `visa-service`) is attached to. Additionally, the Docker Compose file exposes port 5000/udp of the node container, which is the node's docking port, as port 65000/udp *of the host machine*. This makes it possible for adapters on other machines or VMs to connect to the node.

## Policy

The policy source is in `demo.zpl`. Accompanying it is the policy configuration, `demo.zplc`. You may wish to study these to understand...

## Requirements

- x86_64 only.
	- Not ARM (Apple, Pi, etc.) — even with `binfmt` (qemu or Rosetta). 
- Linux as host OS. You must have root privileges.
	- The docker containers run with extra privileges to manipulate network interfaces. Docker on other OSes runs containers in a VM, so this won't work.
- A recent Linux distribution.
	- Ubuntu 24.04 or later; Debian 13 "trixie"; RHEL or Oracle Linux 10.0.
	- Specifically, glibc 2.39 or later.
- Docker Engine (Docker CE) including Docker Compose plugin.
	- The packages from docker.com are recommended. Older versions may not be tested.
- Ability to run VMs (i.e., libvirt/KVM or VirtualBox).

Running the services on separate physical machines, or in other VM setups, is beyond the scope of the demo.

# How to run the demo

To run the demo you need three things:

1. The docker image.
2. The release configuration.
3. The release binaries.

All three components must have the same version. The version is a timestamp
in `YYYYMMDD` format.  For example, `20250919`.

You can find the docker image in the [org-zpr packages area](https://github.com/orgs/org-zpr/packages/container/package/zpr-demo%2Fzprdemo).

The release configuration will be in this repo in a [branch](https://github.com/org-zpr/zpr-demo/branches)
named `demo-VERSION`, where `VERSION` is the version number.

The release binaries can be found in this repo in the [releases](https://github.com/org-zpr/zpr-demo/releases)
section using the same `demo-VERSION` naming convention.



## Run the Demo

Using the correct branch of this repo.

Get the correct binaries from the github *releases* section.


### Get and Launch the Docker container.

cd docker

mkdir -p ...

Then to start the container: `sudo make ZPR_IMAGE_VERSION=latest up`

The docker image starts three containers:
- Node
- Visa service plus adapter
- BAS plus adapter

The node exposes its docking port on the host OS at port `65000`. The config
files for the "cli" (*client*), "admin", and "web" adapters (all in the `release/conf`
directory) are setup with the `node_addr` set to `127.0.0.1:65000` -- that is
<!-- FIXME -->
correct only if connecting from the host OS. Connecting from a VM or whereever
will require overriding that value.


### Start VM, run the web service in it.

```
python3 -m http.server --bind :: 80
```
However: empirically you can STILL do: `curl 192.168.122.27`! Even though only listening on ipv6 as confirmed by `lsof -i -n` ! Probably a linux 4-6 thing

```
root@localhost:~/serve# python3 -m http.server --bind :: 80                                                                                                    
Serving HTTP on :: port 80 (http://[::]:80/) ...
fd5a:5052:1:1::5 - - [10/Nov/2025 21:56:35] "GET / HTTP/1.1" 200 -
::ffff:192.168.122.1 - - [19/Nov/2025 17:15:40] "GET / HTTP/1.1" 200 -
```

FIXME you don't need any "forwarding". Describe what connectivity you actually need

Now create a VM. Make sure UDP traffic can travel between the host and the VM. 

In the VM, start a webserver on port 80

and connect the web adapter:

```bash
sudo ph adapter -l all=DEBUG -c release/conf/adapter-web-conf.toml
```

Note that the `node_addr` setting in the conf file above must be set to
`<HOST-IP-ADDR>:65000`.





### Connect as an Admin

From the host OS you can now connect as the admin by pointing to the
`adapter-admin-conf.toml` file.
Run:

```bash
sudo ph adapter -l all=DEBUG -c release/conf/adapter-admin-conf.toml
```

Leave that running.

And with that you will get privileges to talk to the visa service using
`vs-admin`.
Run:

```bash
../zpr-20251010/vs-admin -s 'https://[fd5a:5052::1]:8182' -c release/conf/auth-ca.crt actors
```

The output will list the actors and their ZPR addresses, which are special IPv6 addresses.
Make a note of the ZPR address of `web.zpr.org`, so you can connect to it later.
Example output:

```
~/zpr-demo$ ../zpr-20251010/vs-admin -s 'https://[fd5a:5052::1]:8182' -c release/conf/auth-ca.crt actors
🐎 found 5 actors
vs.zpr (created: 2025-11-19T17:34:29Z) @ fd5a:5052::1 
node.zpr.org (created: 2025-11-19T17:34:40Z) @ fd5a:5052:90de::1 [node]
bas.zpr.org (created: 2025-11-19T17:34:43Z) @ fd5a:5052:1:1::1 
web.zpr.org (created: 2025-11-19T17:35:35Z) @ fd5a:5052:1:1::2 
admin.zpr.org (created: 2025-11-19T17:36:29Z) @ fd5a:5052:1:1::3 
```



Another interesting `vs-admin` command is `gui` which brings up a
terminal based UI showing some details of the visa service operation.
Just replace `actors` with `gui` in the above line to try it out.

#### Attempt to access Web from Admin

Connected as the admin adapter you cannot access the web service. Refer to the ZPL, `demo.zpl`, to understand why: the policy does not allow it! 
So, for example, this will fail:

```bash
curl http://\[<ZPR-ADDRESS>\]/rfc1.txt
```

This expected failure (attempting to access `web` from `admin`) looks like this:

```
mnestler@zucchini:~/zpr-demo$ curl -v 'http://[fd5a:5052:1:1::2]/'
*   Trying [fd5a:5052:1:1::2]:80...
* connect to fd5a:5052:1:1::2 port 80 from fd5a:5052:1:1::3 port 54486 failed: Connection timed out
* Failed to connect to fd5a:5052:1:1::2 port 80 after 135075 ms: Could not connect to server
* closing connection #0
curl: (28) Failed to connect to fd5a:5052:1:1::2 port 80 after 135075 ms: Could not connect to server
mnestler@zucchini:~/zpr-demo$ 
```

```
2025-11-19T17:40:04.973198Z DEBUG flow_mgmt: link 2: Issuing bind request for (IPv6, fd5a:5052:1:1::3, fd5a:5052::1, 6, 42240, 8182) (is now set PENDING)
2025-11-19T17:40:04.973242Z  INFO zdp: Link 2: sending BindActorAddressRequest for (IPv6, fd5a:5052:1:1::3, fd5a:5052::1, 6, 42240, 8182) with compression mode 0 packet_body size 80
2025-11-19T17:40:05.027853Z DEBUG flow_mgmt: Bind of (IPv6, fd5a:5052:1:1::3, fd5a:5052::1, 6, 42240, 8182) succeeded: 3
2025-11-19T17:40:05.077548Z DEBUG zdp: Link 2: handling mgmt message type BindActorAddressRequest seq_num 80
2025-11-19T17:40:05.077610Z DEBUG zdp: Link 2: handlers.handle_bind_actor_address_request -- five_tuple (IPv6, fd5a:5052::1, fd5a:5052:1:1::3, 6, 8182, 42240)
2025-11-19T17:40:53.233088Z DEBUG flow_mgmt: link 2: Issuing bind request for (IPv6, fd5a:5052:1:1::3, fd5a:5052:1:1::2, 6, 54486, 80) (is now set PENDING)
2025-11-19T17:40:53.233137Z  INFO zdp: Link 2: sending BindActorAddressRequest for (IPv6, fd5a:5052:1:1::3, fd5a:5052:1:1::2, 6, 54486, 80) with compression mode 0 packet_body size 80
2025-11-19T17:40:53.281962Z DEBUG flow_mgmt: Bind of (IPv6, fd5a:5052:1:1::3, fd5a:5052:1:1::2, 6, 54486, 80) failed: policy error
```
(last 3 lines repeating)



So kill the admin adapter on the host (ctrl-C), and move on to the next section to re-connect as the "cli" adapter.

(NOTE TO ADD TO THE README: of course, "cli" and "admin" could both be part of the ZPR net at once, but not from the same host (maybe??????)


### Connect as a client

```bash
sudo ph adapter -l all=DEBUG -c release/conf/adapter-cli-conf.toml
```

While that's running, now try the above `curl` command again and it should work.

[TODO explain the commands! And the help for them]


# Troubleshooting

...

firewall if any

nc (netcat-openbsd) udp both ways between: node container and web VM; check TCP HTTP between VS and BAS (check my Slack)



tcpdump in namespace howto. (desperate)


ip route show (?)
ip link
ip addr

example output of vs-admin, which shows what we will need for the curl command!


(another note for readme: in this demo, zpr addresses are ipv6, but nothing else is!)

troubleshooting with zpdump (takes demo.bin) (see text file)

opens a port 65000/udp on your HOST

What success should look like


docker ps

mkdir



## What error messages are normal

- node: `cert failed signature verification`
- node: ... `has unverified name [commonName =` ...

```
node          | 2025-11-19T17:47:40.288164Z  WARN key_mgmt: cert failed signature verification
node          | 2025-11-19T17:47:40.293849Z  WARN link_state: Link 7 has unverified name [commonName = "cli.zpr.org"]
```

- node: `Ignoring unexpected timeout in state Active`

```
node          | 2025-11-19T17:47:55.619098Z ERROR link_state: error handling timeout: Invalid operation: Ignoring unexpected timeout in state Active
```


## Known issues

If you leave it a long time, have to restart. Otherwise, e.g. (note the last line):

```
node          | 2025-11-19T17:23:57.702820Z  INFO link_state: Link 8 started.  Keying in progress
node          | 2025-11-19T17:23:57.702920Z  INFO peer_mgmt: Successfully started tether with 129.6.7.1:56916.  Assigned ID 8
node          | 2025-11-19T17:23:57.709753Z  WARN key_mgmt: cert failed signature verification
node          | 2025-11-19T17:23:57.717576Z  WARN link_state: Link 8 has unverified name [commonName = "admin.zpr.org"]
visa-service  | 2025-11-19T17:23:57.841Z        info    request visa    {"peer": "fd5a:5052:90de::1", "src_tether_addr": "fd5a:5052:0:aaa:0:166:d943:bdc6", "pkt_len": 80}
visa-service  | 2025-11-19T17:23:57.841Z        info    allowing visa request from anon to auth service, overriding policy      {"service": "AuthService"}
visa-service  | 2025-11-19T17:23:57.841Z        info    visa denied, dest actor auth has expired
```

# Further reading

## ZPR Documentation

- [ZPR RFCs](https://github.com/org-zpr/zpr-rfcs). Official overview.

Most other documentation, and source code, is not public yet. However, the following is available now:

- [ZPLC - ZPL Configuration](https://github.com/org-zpr/zpr-compiler/blob/main/README_ZPLC.md). Describes .zplc files.

- Each binary in the [zpr-demo release](https://github.com/org-zpr/zpr-demo/releases) tarball outputs usage information with the `--help` option, and for subcommands as applicable.
	- **`ph`**
	- **`vs-admin`**
	- **`zpdump`**
	- **`zplc`**
- The following additional binaries can be found inside the Docker image. These too output usage information with `--help`.
	- **`bas`**
	- **`vservice`**
	

<!-- zpr-core is still not public as of 2025-11-19

- zpr-core
	- [Readme / How to setup a ZPRnet](https://github.com/org-zpr/zpr-core/blob/main/README.md). Your next steps to go beyond this demo.
	- [Packet Walk](https://github.com/org-zpr/zpr-core/blob/main/packet_walk.md)

## Previous demos

These old demos and their readmes are still informative.

- [Milestone 2](https://github.com/org-zpr/zpr-core/blob/main/examples/milestone2/README.md)
- [Milestone 3](https://github.com/org-zpr/zpr-core/blob/main/examples/milestone3/README.md)
- [Milestone 4](https://github.com/org-zpr/zpr-core/blob/ed4b971cdc2cdf92e0f8d73f14ecb10336a0fbdb/examples/milestone4/README.md) (precursor to the present `zpr-demo`)

-->