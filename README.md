# home-server-setup
Setup Files for Home Server Instances and Satellites for Smart Home

The initial set of files assumes an Ubuntu 22.04 LTS (Jammy) or compatible system. The scripts require
ansible and git. You can also fork the repo to tweak scripts for other systems

To install them on Jammy, use the following commands:

`sudo apt update`
`sudo apt install ansible git`

To set up a Rhasspy satellite computer from a basic Ubuntu installation, complete the following steps:

`sudo ansible-pull -U https://github.com/doctorjei/home-server-setup.git -i localhost`

This playbook sets up Portainer, Syncthing, and Rhasspy on standard ports - Portainer for management,
Syncthing for backups, and of course Rhasspy for the saltellite service. :)

The playbook also creates a 'podman' user for containers and the regular / sudoer 'ubuntu' user.
