# sidedoor

Backdoor using a reverse SSH tunnel on Debian/Ubuntu systems

sidedoor maintains a reverse SSH tunnel to provide a backdoor.
sidedoor can be used to remotely control a device behind a NAT.

The sidedoor user has full root access configured in /etc/sudoers.d.

## Howto

 1. Install sidedoor. For now, `sudo dpkg -i sidedoor*.deb`.
    You can build a package by running `dpkg-buildpackage -us -uc -b`.
 2. Optionally, lock down the local SSH server by disabling password
    authentication (`ChallengeResponseAuthentication no` and
    `PasswordAuthentication no`) and listening only on localhost
    (`ListenAddress ::1` and `ListenAddress 127.0.0.1`) in
    /etc/ssh/sshd_config. Then restart or reload sshd, e.g.,
    `sudo service ssh reload`.
 3. Configure REMOTE_SERVER and TUNNEL_PORT in /etc/default/sidedoor.
 4. Install an SSH private key to access the remote server in
    /etc/sidedoor/id_rsa. The corresponding public key will
    need to be included in the remote user's ~/.ssh/authorized_keys file.
 5. Install SSH public key(s) to control access to the local sidedoor user
    in /etc/sidedoor/authorized_keys.
 6. Restart sidedoor service, e.g., `sudo service sidedoor restart`.
 7. Optionally, modify ssh_config_example and include it in a client's
    ~/.ssh/config file to easily access the tunnelled backdoor.

## License

Copyright 2015 Dara Adib.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
