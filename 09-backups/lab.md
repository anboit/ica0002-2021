# Lab 9

The goal of this and the next lab is to configure the backups for your infrastructure.

In this lab we'll prepare the backup documentation and set up the connection with the backup server.

In the next lab we will set up automatic backups themselves, and improve the documentation
accordingly.


## Current infrastructure

This is what we have built so far:

![](lab-9-infra.png)

Teachers have also prepared a backup server for you. It is located in the same network with all your
virtual machines, so consider it an _on-site_ resource. See task 1 for more details.

For simplicity you can assume that all the data from this server is also copied to another storage
outside of our infrastructure perimeter, and whatever backup you'll upload to this server will also
get an _off-site_ copy automatically -- so you don't need to worry about the `-1-` part of the
`3-2-1` rule.


## Task 1

Configure your DNS server to resolve the backup server name in your domain.

Backup server IP address is 192.168.42.156. Note that this address may change in future, so do not
hard-code it but store as the Ansible variable.

Once done, each of your virtual machines should be able to resolve backup server name. You can check
it by running this command on any of your VMs (replace `foo.bar` with your domain you have set up in
[lab 5](../05-dns-server/lab.md)):

    host backup.foo.bar.

You should receive something like this in response:

    backup.foo.bar has address 192.168.42.156


## Task 2

Backups will be uploaded from your managed hosts to the backup server over SSH. More details will be
provided in the next lab, but first we'll need to get this SSH connection working. Of course we'll
use SSH keys for user authentication.

For that we'll need a separate user account on every managed host, and also a few more SSH keypairs.

**Remember -- different users should have different private SSH keys!**

Add Ansible role `backup` with the tasks that will
 1. Create a user named `backup` on every managed host
 2. Generate a new SSH key pair for this user

Requirements:
 1. User `backup` home directory should be `/home/backup`
 2. User `backup` public SSH key should be located at `/home/backup/.ssh/id_rsa.pub`

Note that there may be user named `backup` created already on the managed host so you will need to
modify it to match the requirements above. Your Ansible code will be the same for both cases.

Make sure to generate a **new** keypair for the user `backup`; Ansible module
[user](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html)
has an option to do it automatically -- check the module documentation.

**DO NOT upload YOUR OWN SSH private key (used for GitHub) to managed hosts!**

If you feel that SSH keys are still unclear topic for you please refer to
[lecture 2 slides](../02-web-server) that covered it.

Add this role to `Init` play created in [lab 4](../04-troubleshooting/lab.md) and run Ansible to
apply the changes:

    ansible-playbook infra.yaml

Once the user is created and SSH keypair is generated for it the public key will be authorized on
the backup server automatically within some time (usually 15 minutes). A separate account will be
created for your backups, account name will match your GitHub username.

You can focus on the next task so far, and check the result later.

Once the key is authorized on the backup server the `backup` user you've just created should be able
to log in there. You can test it by running this command manually on the managed host:

    sudo -u backup ssh <your-github-username>@backup.<your-domain> id

Example for GitHub username `elvis` and domain `demo.tld` (modify for your names accordingly):

    sudo -u backup ssh elvis@backup.demo.tld id

Note that user account names are different:
 - on your managed host (which is only yours) it is `backup`
 - on the backup server (which is a shared resource) it matches your GitHub username

If the key was authorized successfully you should get something like this in response:

    uid=1042(elvis) gid=1042(elvis) groups=1042(elvis)

Numbers will be different but the username should match your GitHub username.

If you are getting this error:

    Permission denied (publickey).

then please
 1. make sure you are running the command from the correct server: your managed host
 2. make sure you are running the command as the correct user: `backup`
 3. make sure that user `backup` home directory is `/home/backup`: can be checked with `echo ~backup`
 4. make sure that `/home/backup/.ssh/id_rsa.pub` file is there
 5. if the problem is still there, contact the teachers for help


## Task 3

Create a free-text file named `backup_sla.md` in the root of your Ansible repository.

In that file describe the backup approach for:
 - Database servers
 - InfluxDB
 - Ansible repository

This document should contain the information about:
 - Backup coverage -- what is backed up and what is not
 - RPO (recovery point objective)
 - Versioning and retention -- how many backup versions are stored and for how long
 - Usability checks -- how is backup usability verified
 - Restoration criteria -- when should backup be restored
 - RTO (recovery time objective)

Note that you don't need to backup _each and every_ service. In case of disaster recovery it makes
more sense to re-create some service from scratch than restore it from the backup (Nginx, AGAMA,
Grafana, Bind). But for other services backups are must (MySQL and InfluxDB contain data that cannot
be restored with Ansible).

If you are missing some information to complete backup documentation (list of responsible IT staff,
amount of data, value of information, acceptable data loss) -- make it up! Be creative, but try to
keep it adequate and somewhat related to real life.


## Expected result

Your repository contains these files and directories:

    ansible.cfg
    backup_sla.md
    infra.yaml
    roles/backup/tasks/main.yaml

Your repository also contains all the required files from the previous labs.

Backup user is configured as required on every your managed host by running this command:

    ansible-playbook infra.yaml
