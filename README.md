# sidedoor

sidedoor maintains an SSH connection or tunnel
with a shell script daemon.

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

sidedoor is packaged for Debian-based systems like Raspbian and Ubuntu,
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
 2. Edit SSH configuration files under `/etc/sidedoor`.
    - **`id_rsa`**: SSH private key to access the remote server.
      Can use `ssh-keygen` to create this key
      (press y when prompted to overwrite the existing file):

          sudo ssh-keygen -t rsa -N '' -f /etc/sidedoor/id_rsa

      The corresponding public key `id_rsa.pub` will need to be included in
      the remote user's `~/.ssh/authorized_keys` file.
    - **`known_hosts`**: SSH host key of the remote server.
 3. Optionally, grant remote access to the local sidedoor user by adding
    SSH public key(s) to the file `/etc/sidedoor/authorized_keys`.
    `/etc/sidedoor/authorized_keys` is a symlink to
    `~sidedoor/.ssh/authorized_keys`.
    The `sidedoor-sudo` package, if installed, provides full root access
    to this user.

 4. Restart the sidedoor service to apply changes.

        sudo service sidedoor restart

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

## Comparison with autossh

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

## Other alternatives

sidedoor is intended as a lightweight solution to tunneling ports
with minimal dependencies. Some more featured alternatives include:

 * [OpenVPN](https://en.wikipedia.org/wiki/OpenVPN)
 * [PageKite](https://github.com/pagekite/PyPagekite/)
 * [ssh_tunnel](http://sshtunnel.sourceforge.net/)

## License

Copyright 2015-2016 Dara Adib.

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
[portforwarding]: https://blog.trackets.com/2014/05/17/ssh-tunnel-local-and-remote-port-forwarding-explained-with-examples.html
