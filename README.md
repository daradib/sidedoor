# sidedoor

sidedoor maintains an SSH connection or tunnel
with a shell script daemon.

![sidedoor tunneling](https://quietapple.org/dl/sidedoor.svg)

The primary use case is maintaining a remote port forward
to the local SSH server (or another port). Thus, the local
device can be accessed without using incoming connections
that may be blocked by a NAT or firewall or otherwise
impractical with mobile devices.

SSH clients can connect to the device via the reverse SSH proxy
that sidedoor tunnels to. This proxy server can be untrusted
and run by a third party or cloud service.

sidedoor enables SSH keepalives and retries SSH with
exponential backoff. In order to reconnect as soon as possible,
it resets the backoff when a network interface is brought up
(or changed).

Other use cases:

 * Access a web application behind a NAT by remote forwarding the
   local web server (e.g., port 80).
   A remote server can host a reverse proxy to the web application
   and handle SSL/TLS termination.
 * Stay connected to office network services behind an
   SSH [bastion host](https://en.wikipedia.org/wiki/Bastion_host)
   by local forwarding them.
 * [Melt Evil Corp's tape backups][mrrobot]
   by remotely controlling a Raspberry Pi (not recommended!).

**Are you using sidedoor?**
Bugs reports, feature requests - please open an issue!
Pull requests are welcome.

## Installation

sidedoor is packaged for Debian and Debian-based systems like
Raspbian, Ubuntu, and VyOS/[EdgeOS][edgeos],
but should work in any POSIX environment with an (OpenSSH) SSH client.

If sidedoor is in your distribution repositories (Debian 9+, Ubuntu 17.04+),
simply install it with your package manager.

    sudo apt install sidedoor

Otherwise, you can manually download debs from the
[Releases page](https://github.com/daradib/sidedoor/releases).

To grant the sidedoor user full root access,
install the `sidedoor-sudo` package.

## Configuration

The remote server and port forwards are configured in `/etc/default/sidedoor`.
SSH configuration files are located in the `/etc/sidedoor` directory.

 1. Configure `REMOTE_SERVER` and `OPTIONS` in `/etc/default/sidedoor`.
    For some arguments to pass in `OPTIONS`, see the blog post
    [Local and Remote Port Forwarding Explained With Examples][portforwarding]
    and the [`ssh` man page](https://linux.die.net/man/1/ssh).
 2. Create the `id_rsa` / `id_rsa.pub` ssh key pair under `/etc/sidedoor`:
    You can use `ssh-keygen` to create this key
    (press y when prompted to overwrite the existing file):

          sudo ssh-keygen -t rsa -N '' -f /etc/sidedoor/id_rsa

 3. Add the remote server's ssh host key to sidedoor's known hosts
    (replace `REMOTE_SERVER` with your actual remote server's address):

          ssh-keyscan REMOTE_SERVER | sudo tee /etc/sidedoor/known_hosts

 4. Optionally, grant remote access to the local sidedoor user by unlocking
    the account with `usermod -U sidedoor` and adding
    SSH public key(s) to the file `/etc/sidedoor/authorized_keys`.
    `/etc/sidedoor/authorized_keys` is a symlink to
    `~sidedoor/.ssh/authorized_keys`.
    The `sidedoor-sudo` package, if installed, provides full root access
    to this user.

 5. Restart the sidedoor service to apply changes.

        sudo service sidedoor restart

## Remote Server Configuration

1. Create a user that can only perform port forwardings and can only log in via
   public key:

          adduser --shell /usr/sbin/nologin --disabled-password --gecos x sidedoor-portfw

2. Copy `/etc/sidedoor/id_rsa.pub` from your local pc to `/home/sidedoor-portfw/.ssh/authorized_keys`
3. Make sure that `ClientAliveInterval 30` is set in `/etc/ssh/sshd_config`. This makes sure that a hung ssh session is terminated after
   some time and the remote port is freed up when sidedoor reconnects. Any non-zero value works, but `30` is reasonable.

## Recommendations

 * Lock down the local SSH server by editing `/etc/ssh/sshd_config`.
   - Disable password authentication
     (`ChallengeResponseAuthentication no` and `PasswordAuthentication no`).
   - Limit daemon to only listen on localhost
     (`ListenAddress ::1` and `ListenAddress 127.0.0.1`).
   - To apply changes, restart or reload `sshd`, e.g.,
     `sudo service ssh reload`.
 * Modify the `ssh_client_config_example` file and include it in a client's
   `~/.ssh/config` file to easily access the tunneled SSH server
   with `ssh`, `scp`, `rsync`, etc.

## Alternatives

sidedoor is intended as a lightweight solution to tunneling ports
with minimal dependencies, but there are some alternatives with more
features.

### Tor hidden service

Tor provides anonymity to servers run as [hidden services][hidden-service],
but also handles NAT traversal.

Advantages:

 * Metadata, including the IP address of the local device
   and its connection state (on/off), is less exposed to an intermediary
   like the reverse SSH proxy.

Disadvantages:

 * Tor must be installed and running on both the local device and clients.
 * Tor has higher latency so terminal feedback (input echo) is slow.

On both the device and clients, install Tor.

    sudo apt install tor

On the device that is being exposed, edit [`/etc/tor/torrc`][torrc]
to create a hidden service on port 22.

    HiddenServiceDir /var/lib/tor/sshd/
    HiddenServicePort 22 127.0.0.1:22
    HiddenServiceAuthorizeClient stealth client

Replace "client" with a comma-separated list of client names to
generate multiple authorization secrets.

Then reload Tor and get the onion hostname and authorization data.

    sudo service tor reload
    sudo cat /var/lib/tor/sshd/hostname

On clients, edit [`/etc/tor/torrc`][torrc]
to add the onion hostname and authorization data seen in the `hostname` file.

    HidServAuth <hostname>.onion <secret>

Then reload Tor and run `torsocks ssh <hostname>.onion` or set `ProxyCommand`
in the `~/.ssh/config` file.

    ProxyCommand torsocks nc <hostname>.onion 22

### autossh

[autossh](http://www.harding.motd.ca/autossh/), like sidedoor,
starts `ssh` and restarts it as needed.

Some differences include:

 * sidedoor is a minimalistic shell script daemon.
   autossh is a more extensive and configurable C program.

 * sidedoor enables SSH keepalives
   (`ServerAliveInterval` and `ServerAliveCountMax`),
   which are available in modern versions of OpenSSH.
   autossh monitors `ssh` by sending data through
   a loop of port forwards (this feature predates SSH keepalives),
   though this can be disabled with the `-M 0` option.

 * sidedoor is intended to run automatically as a service,
   so the package includes init/systemd scripts and config files.
   autossh does not include an init/systemd script
   (Debian bug [#698390](https://bugs.debian.org/698390)).

 * sidedoor disables remote commands and pseudo-tty allocation.
   For interactive use, consider autossh with SSH keepalives
   or [Mosh](https://mosh.org/).

 * sidedoor always retries if `ssh` exits with a non-zero exit status.
   autossh does not retry if `ssh` exits too quickly on the first attempt,
   which can happen when network connectivity or DNS resolution
   is broken, particularly on mobile devices.
   Both sidedoor and autossh have retry backoff logic.

 * sidedoor resets retry backoff when a network interface is brought up,
   to attempt to reconnect as soon as possible, by receiving SIGUSR1
   from an `if-up.d` script. autossh does not have network state hooks.

### Other alternatives

 * [OpenVPN](https://en.wikipedia.org/wiki/OpenVPN)
 * [PageKite](https://github.com/pagekite/PyPagekite/)
 * [ssh_tunnel](http://sshtunnel.sourceforge.net/)

## License

Copyright 2015-2017 Dara Adib.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <https://www.gnu.org/licenses/>.

[mrrobot]: https://www.forbes.com/sites/abigailtracy/2015/07/15/hacking-the-hacks-mr-robot-episode-four-sam-esmail/
[edgeos]: https://help.ubnt.com/hc/en-us/articles/205202560-EdgeMAX-Add-other-Debian-packages-to-EdgeOS
[portforwarding]: https://blog.trackets.com/2014/05/17/ssh-tunnel-local-and-remote-port-forwarding-explained-with-examples.html
[hidden-service]: https://www.torproject.org/docs/tor-hidden-service.html.en
[torrc]: https://www.torproject.org/docs/tor-manual.html.en
