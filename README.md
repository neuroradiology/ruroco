[![build](https://github.com/beac0n/ruroco/actions/workflows/rust.yml/badge.svg)](https://github.com/beac0n/ruroco/actions)
[![release](https://img.shields.io/github/v/release/beac0n/ruroco?style=flat&labelColor=1C2C2E&color=C96329&logo=GitHub&logoColor=white)](https://github.com/beac0n/ruroco/releases)
[![codecov](https://codecov.io/gh/beac0n/ruroco/graph/badge.svg?token=H7ABBHYYWT)](https://codecov.io/gh/beac0n/ruroco)

# ruroco - run remote command

Ruroco is a tool to run pre-defined commands on a remote server, using the UDP protocol to hide the existence of the
service from adversaries, making the service on the server "invisible".

# use case

If you host a server on the web, you know that you'll get lots of brute-force attacks on (at least) the SSH port of your
server. While using good practices in securing your server will keep you safe from such attacks, these attacks are quite
annoying (filling up logs) and even if you secured your server correctly, you will still not be 100% safe, see
https://www.schneier.com/blog/archives/2024/04/xz-utils-backdoor.html or
https://www.qualys.com/2024/07/01/cve-2024-6387/regresshion.txt

Completely blocking all traffic to all ports that do not have to be open at all times can reduce the attack surface.
But blocking the SSH port completely will make SSH unusable for that server.

This is where ruroco comes in. Ruroco can execute a command that opens up the SSH port for just a short amount of time,
so that you can ssh into your server. Afterward ruruco closes the SSH port again. To implement this use case with
ruroco, you have to use a configuration similar to the one shown below:

```toml
address = "0.0.0.0:8080"  # address the ruroco serer listens on, if systemd/ruroco.socket is not used
config_dir = "/etc/ruroco/"  # path where the configuration files are saved

[commands]
# open ssh, but only for the IP address where the request came from
open_ssh = "ufw allow from $RUROCO_IP proto tcp to any port 80"
# close ssh, but only for the IP address where the request came from
close_ssh = "ufw delete allow from $RUROCO_IP proto tcp to any port 80"
```

If you have configured ruroco on server like that and execute the following client side command

```shell
ruroco-client send --address host.domain:8080 --private-pem-path /path/to/ruroco_private.pem --command open_ssh --deadline 5
```

the server will validate that the client is authorized to execute that command by using public-key cryptography (RSA)
and will then execute the command defined in the config above under "open_ssh". The `--deadline` argument means that the
command has to be started on the server within 5 seconds after executing the command.

This gives you the ability to effectively only allow access to the SSH port, for only the IP that the UDP packet was
sent from, if you want to connect to your server. Of course, you should also do all the other security hardening tasks
you would do if the SSH port would be exposed to the internet.

You can define any number of commands you wish, by adding more commands to configuration file.

# setup

download binaries from the [releases page](https://github.com/beac0n/ruroco/releases) or build them yourself by running

```shell
make release
```

you can find the binaries in `target/release/client`, `target/release/server` and `target/release/commander`

## client

### self-build

See make goal `install_client`. This builds the project and copies the client binary to `/usr/local/bin/ruroco-client`

### pre-build

Download the `client-v<major>.<minor>.<patch>-x86_64-linux` binary from https://github.com/beac0n/ruroco/releases/latest
and move it to `/usr/local/bin/ruroco-client`

## server

### self-build

See make goal `install_server`, which

- Builds the project
- Copies the binaries to `/usr/local/bin/`
- Adds a `ruroco` user if it does not exist yet
- Copies the systemd service files and config files to the right places
- Assigns correct file permissions to the systemd and config files
- Enables and starts the systemd services
- After running the make goal, you have to
    - generate a RSA key and copy it to the right place
    - setup the `config.toml`

### pre-build

1. Download the `server-v<major>.<minor>.<patch>-x86_64-linux` and `commander-v<major>.<minor>.<patch>-x86_64-linux`
   binaries from https://github.com/beac0n/ruroco/releases/latest and move them to
   `/usr/local/bin/ruroco-server` and `/usr/local/bin/ruroco-commander`.
2. Make sure that the binaries have the minimal permission sets needed:
    1. `sudo chmod 500 /usr/local/bin/ruroco-server`
    2. `sudo chmod 100 /usr/local/bin/ruroco-commander`
    3. `sudo chown ruroco:ruroco /usr/local/bin/ruroco-server`
3. Create the `ruroco` user: `sudo useradd --system ruroco --shell /bin/false`
4. Install the systemd service files from the `systemd` folder: `sudo cp ./systemd/* /etc/systemd/system`
5. Create `/etc/ruroco/config.toml` and define your config and commands, e.g.:
    ```toml
    address = "127.0.0.1:8080"  # address the ruroco serer listens on, if systemd/ruroco.socket is not used
    config_dir = "/etc/ruroco/"  # path where the configuration files are saved
    
    [commands]
    # commands will be added here - see configuration chapter
    ```
    1. Make sure that the configuration file has the minimal permission set
       needed: `sudo chmod 400 /etc/ruroco/config.toml`
6. Since new systemd files have been added, reload the daemon: `sudo systemctl daemon-reload`
7. Enable the systemd services:
    1. `sudo systemctl enable ruroco.service`
    2. `sudo systemctl enable ruroco-commander.service`
    3. `sudo systemctl enable ruroco.socket`
8. Start the systemd services
    1. `sudo systemctl start ruroco-commander.service`
    2. `sudo systemctl start ruroco.socket`
    3. `sudo systemctl start ruroco.service`

# configuration

## generate and deploy rsa key

- run `ruroco-client gen` to generate two files: `ruroco_private.pem` and `ruroco_public.pem`
- move `ruroco_public.pem` to `/etc/ruroco/ruroco_public.pem` on server
- save `ruroco_private.pem` to `~/.config/ruroco/ruroco_private.pem` on client

## update config

Add commands to config `/etc/ruroco/config.toml` on server. The new config file **could** look like this:

```toml
address = "127.0.0.1:8080"  # address the ruroco serer listens on, if systemd/ruroco.socket is not used
config_dir = "/etc/ruroco/"  # path where the configuration files are saved

[commands]
# open ssh, but only for the IP address where the request came from
open_ssh = "ufw allow from $RUROCO_IP proto tcp to any port 80"
# close ssh, but only for the IP address where the request came from
close_ssh = "ufw delete allow from $RUROCO_IP proto tcp to any port 80"
```

# security

A lot of thought has gone into making this tool as secure as possible:

- The client sends a UDP packet to the server, to which the server never responds. So port-scanning does not help an
  adversary.
- The server only holds the public key. The client uses the private key to send an encrypted packet.
- Each request that is sent holds the current timestamp and the command that the server should execute.
  This encrypted packet is only valid for a configurable amount of time.
- On the server, the service that received the UDP package has as little OS rights as possible (restricted by systemd).
  After validating the data, the service that received the UDP packet (server) instructs another service (commander) to
  execute the command. So even if the server service is compromised, it can't do anything, because it's rights are
  extremely limited.
- Each packet can only be sent once and will be blacklisted on the server.
- (WIP) To make the service less vulnerable against DoS attacks ...

# architecture

## overview

The service consists of three parts:

- `client`
    - binary that is executed on your local host
- `server`
    - service that runs on a remote host where you wish to execute the commands on
    - exposed to the internet
    - has minimal rights to receive and decrypt data and to communicate with the commander
- `commander`
    - daemon service that runs on the same host as the server
    - not exposed to the internet
    - has all the rights it needs to run the commands that are passed to it

<!-- created with https://asciiflow.com/#/ -->

```text
┌────────────────┐ ┌────────────────┐
│                │ │                │
│   ┌────────┐   │ │ ┌────────────┐ │
│   │ Client ├───┼─┤►│   Server   │ │
│   └────────┘   │ │ └─────┬──────┘ │
│                │ │       │        │
│                │ │ ┌─────▼──────┐ │
│                │ │ │  Commander │ │
│                │ │ └────────────┘ │
│   Local Host   │ │   Remote Host  │
└────────────────┘ └────────────────┘
```

## execution

Whenever a user sends a command via the client, the following steps are executed

1. client concatenates the current timestamp (in nanoseconds) with the command name (e.g. "default"), encrypts data with
   the private key and sends the encrypted data via UDP to the server
2. server receives the UDP package (does **not** answer), decrypts it with the public key and validates its content
3. if the content is valid, the server sends the command name to the commander. If the content is invalid an error
   message is logged
4. commander receives the command name and executes the command if the command is defined in the configuration

```text
     ┌─────────┐            ┌─────────┐              ┌───────────┐              
     │ Client  │            │ Server  │              │ Commander │              
     └────┬────┘            └────┬────┘              └─────┬─────┘              
          │                      │                         │                    
          │ Encrypt and send     │                         │                    
          ├─────────────────────►│                         │                    
          │ data via UDP         │                         │                    
          │                      │ Decrypt and validate    │                    
          │                      ├─────────────┐ data      │                    
          │                      │             │           │                    
          │                      │◄────────────┘           │                    
          │                      │                         │                    
          │                      │                         │                    
          │                      │                         │                    
┌────┬────┼──────────────────────┼─────────────────────────┼───────────────────┐
│alt │    │                      │                         │                   │
├────┘    │                      │ Log error               │                   │
│if is    │                      ├─────────────┐           │                   │
│invalid  │                      │             │           │                   │
│         │                      │◄────────────┘           │                   │
│         │                      │                         │                   │
├─  ──  ──┤ ──  ──  ──  ──  ──  ─┤  ──  ──  ──  ──  ──  ── ├──  ──  ──  ──  ── │
│else     │                      │                         │                   │
│         │                      │ Send command name       │                   │
│         │                      ├────────────────────────►│                   │
│         │                      │                         │                   │
│         │                      │                         │ Check if command  │
│         │                      │                         │ is valid          │
│         │                      │                         ├───────────────┐   │
│         │                      │                         │               │   │
│         │                      │                         │◄──────────────┘   │
│         │                      │                         │                   │
│         │            ┌────┬────┼─────────────────────────┼───────────────────┤
│         │            │alt │    │                         │ Execute command   │
│         │            ├────┘    │                         ├───────────────┐   │
│         │            │if is    │                         │               │   │
│         │            │valid    │                         │◄──────────────┘   │
│         │            │         │                         │                   │
│         │            ├─  ──  ──┤ ──  ──  ──  ──  ──  ──  ├─  ──  ──  ──  ──  │
│         │            │else     │                         │Log Error          │
│         │            │         │                         ├───────────────┐   │
│         │            │         │                         │               │   │
│         │            │         │                         │◄──────────────┘   │
│         │            │         │                         │                   │
└─────────┴────────────┴─────────┴─────────────────────────┴───────────────────┘
```
