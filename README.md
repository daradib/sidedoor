# sidedoor

sidedoor maintains an SSH connection or tunnel. Roughly a poor man's VPN.

Example use cases:

 * Control a device behind a NAT by remote forwarding the
   local SSH server (e.g., port 22).
   SSH clients can connect to the device via a remote server
   with end-to-end encryption.
 * Access a web application behind a NAT by remote forwarding the
   local web server (e.g., port 80).
   A remote server can host a reverse proxy to the web application
   and handle SSL/TLS termination.
 * Stay connected to office network services behind an
   SSH [bastion host](https://en.wikipedia.org/wiki/Bastion_host)
   by local forwarding them. No need to use a VPN.
 * [Melt Evil Corp's tape backups](https://www.forbes.com/sites/abigailtracy/2015/07/15/hacking-the-hacks-mr-robot-episode-four-sam-esmail/)
   by remotely controlling a Raspberry Pi (not recommended!).

**Are you using sidedoor?** Add yourself to the
[Users wiki page](https://github.com/daradib/sidedoor/wiki/Users)!
Bugs reports, feature requests - please open an issue!

sidedoor is packaged for Debian-based systems with systemd or upstart.
It has been used on Debian 8 (jessie), Raspbian jessie,
Ubuntu 14.04 LTS (trusty), and Ubuntu 16.04 LTS (xenial).

## Installation

If sidedoor is in your package repositories, simply install it, e.g.,
`sudo apt install sidedoor`.

Otherwise, you will need to build a Debian package and install it.
First, install build dependencies.

    sudo apt install debhelper dh-systemd

Then, from the directory containing this README file, build and install
a package.

    rm -f ../sidedoor*.deb # remove old package builds
    dpkg-buildpackage -us -uc -b
    sudo dpkg -i ../sidedoor_*.deb

Optionally, to grant the sidedoor user full root access,
install the sidedoor-sudo package.

    sudo dpkg -i ../sidedoor-sudo_*.deb

## Configuration

The remote server and port forwards are configured in`/etc/default/sidedoor`.
SSH configuration files are located in the `/etc/sidedoor` directory.

 * Configure `REMOTE_SERVER` and `OPTIONS` in `/etc/default/sidedoor`.
   For some arguments to pass in `OPTIONS`, see the blog post
   [SSH Tunnel - Local and Remote Port Forwarding Explained With Examples](https://blog.trackets.com/2014/05/17/ssh-tunnel-local-and-remote-port-forwarding-explained-with-examples.html)
   and the [ssh man page](https://linux.die.net/man/1/ssh).
 * Create SSH configuration files under `/etc/sidedoor`.
   - `id_rsa`: SSH private key to access the remote server.
     Can be generated with `sudo ssh-keygen -t rsa -f /etc/sidedoor/id_rsa`
     (press enter when prompted for passphrase to leave empty).
     Needs read permission by the sidedoor user or group, e.g.,
     `sudo chown root:sidedoor /etc/sidedoor/id_rsa` and
     `sudo chmod 640 /etc/sidedoor/id_rsa`.
     The corresponding public key `id_rsa.pub` will need to be included in
     the remote user's `~/.ssh/authorized_keys` file.
   - `known_hosts`: SSH host key of the remote server.
 * Optionally, grant remote access to the local sidedoor user by creating
   the file `/etc/sidedoor/authorized_keys` containing SSH public key(s).
   `~sidedoor/.ssh/authorized_keys` is a symlink to
   `/etc/sidedoor/authorized_keys`.
   The sidedoor-sudo package provides full root access to this user.

Restart the sidedoor service to apply changes.

    sudo service sidedoor restart

## Recommendations

 * Lock down the local SSH server by editing `/etc/ssh/sshd_config`.
   - Disable password authentication
     (`ChallengeResponseAuthentication no` and `PasswordAuthentication no`).
   - Limit daemon to only listen on localhost
     (`ListenAddress ::1` and `ListenAddress 127.0.0.1`).
   - To apply changes, restart or reload sshd, e.g.,
     `sudo service ssh reload`.
 * Modify the `ssh_client_config_example` file and include it in a client's
   `~/.ssh/config` file to easily access the tunneled SSH server
   with `ssh`, `scp`, `rsync`, etc.

## Alternatives

sidedoor is meant as a lightweight solution to tunneling ports
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
