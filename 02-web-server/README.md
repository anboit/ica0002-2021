# Lab 2

## Task 1

Update your Ansible inventory file from the [lab 1](../01-intro) and add
the `web_servers` group. It should contain the single host -- your VM
[from your page](http://193.40.156.86/students.html). Example:

	elvis-1  ansible_host=... ansible_port=... ansible_user=...

	[web_servers]
	elvis-1

Update the `infra.yaml` playbook and a single play named `Web server`.
This will set up a web server on your managed host.

This play should run on every hosts from `web_servers` group:

	hosts: web_servers

Note: Ansible logs in to your managed host as unprivileged user `ubuntu` but
most of the following tasks require privilege escalation (should be run as
`root`). Easiest way to achieve that is to set the `become` property in the
play:

	become: yes

See also: https://docs.ansible.com/ansible/latest/user_guide/become.html

Note: you may delete the play `Ansible connection test` from the lab 1.
We don't need it anymore.


## Task 2

Create an Ansible role named `users` that adds two more Linux users on your
managed host and enables SSH user key based authentication for them:
 - `juri` (GitHub username: `hudolejev`)
 - `roman` (GitHub username: `romankuchin`)

Update the `infra.yaml` playbook to apply this role to all your web servers.
Use existing play `Web server`; don't add a new one.

Check out the documentation of these Ansible modules to find out how to do it:
 - [user](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html)
 - [authorized_key](https://docs.ansible.com/ansible/latest/collections/ansible/posix/authorized_key_module.html)


## Task 3

Create an Ansible role named `nginx` that installs and configures Nginx web
server. Use Ansible module
[apt](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html)
for that, and find the exact package name in the Ubutnu package repository:
https://packages.ubuntu.com (in search form select distribution `focal`).

Update the `infra.yaml` playbook to apply this role to all your web servers.
Use existing play `Web server`; don't add a new one.

Do not change the port Nginx is listening on. By default it's 80, and custom
port in your public URL is already set up to forward to port 80 on your managed
host.

Self check:
 - Default Nginx index page should be accessible on the
   [public URL assigned to you](http://193.40.156.86/students.html)


## Task 4

Replace default Nginx index page with a custom one.

First, find out where is the HTML directory located on server. This information
can be found in one of the the Nginx configuration files in the
`/etc/nginx/sites-enabled` directory -- search for `server_root` directive.

Note: you can check these files manually using `cat` or `less`; you don't have
to use Ansible for that.

Then, use Ansible module
[copy](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)
to replace the default HTML page with [this one](./index.html);
replace `VM_NAME` with your VM name.

Self check:
 - Your modified Nginx index page should be accessible on the
   [public URL assigned to you](http://193.40.156.86/students.html)


## Expected result

Your repository contains these files:

 - `ansible.cfg`
 - `hosts`
 - `infra.yaml`
 - `roles/nginx/files/index.html`
 - `roles/nginx/tasks/main.yaml`
 - `roles/users/tasks/main.yaml`

Nginx is installed and configured on empty machine by running exactly this
command exactly once:

	ansible-playbook infra.yaml

Users `juri` and `roman` can SSH to that machine using their SSH keys.

Running the same command again does not make any changes on managed host.

Custom web page that you have uploaded on managed host is served on
[your public URL](http://193.40.156.86/students.html).
