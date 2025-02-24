# Lab 4

In this lab we'll improve setup from the [lab 3](../03-web-app/lab.md) by adding a separate database
server for our app. We'll also learn how to use Ansible variables and Vault.

**Important!**

This and some following labs have tasks to handle secrets (passwords, keys etc.).
Make sure **not to commit plain text secrets to GitHub!**

Should you make this mistake, change the secret at once, encrypt it propperly (details are provided
below and in lecture slides) and push the next Git commit that overwrites the secret. Leaked secret
value still remains in the Git history -- but as you have changed it it's not a big problem anymore.

Note that your solution is not accepted if your Git history contains secrets that are still valid
(can be used to access your running services).


## Task 1: Set up Ansible Vault

First, create a master password.
It will be used to encrypt and decrypt other secrets in your Ansible repository.
Use any password generator that you like.

Then, save this password to a file **outside of your Ansible repository**.
One good choice is `~/.ansible/vault_password` -- but you can use other path if you want.
This file should contain just password, nothing more:

    $ cat ~/.ansible/vault_password
    y0ur_p4ssw0rd_h3r3

Make sure this file is readable only to you.
You can use `chmod` command to set the file permissions:

    chmod 600 ~/.ansible/vault_password

Finally, configure the Ansible to read the Vault password from this file --
update `ansible.cfg` and add the following setting to the `defaults` section:

    vault_password_file = ~/.ansible/vault_password

(modify as needed if you use a different path).

You can verify that you did everything correctly by running these commands in the root of your
Ansible repository. Run them exactly as written, without any additional parameters.

Create a plain text file:

    echo WORKS > ansible-vault-test.txt

Encrypt this file; this should print 'Encryption successful':

    ansible-vault encrypt ansible-vault-test.txt

Decrypt this file and print the decrypted text; this should print 'WORKS':

    ansible-vault view ansible-vault-test.txt

If that worked, delete the file, we don't need it anymore:

    rm ansible-vault-test.txt

If these commands run without any errors and you could decrypt the file --
you're all set and good to go.


## Task 2: Update Ansible inventory

For this lab you'll need two virtual machines:
one for an app set up in the previous lab, and another for a standalone database server.

> Two empty machines are created for you if you have completed the lab 3 successfully, and can be
> found [on your page](http://193.40.156.86/students.html) as usually.
>
> If you haven't completed lab 3 yet -- please do that first, wait for machines to be created, and
> then continue with this lab.
>
> If you have completed the lab 3 and still don't see your machines, or see only one --
> please contact the teachers.

Update your Ansible inventory file and make sure machine connection parameters are correct.

Add the new host group `db_servers` to your inventory file.
Add the new machine named "<yourname-2>" there.
Leave the old machine as member of the existing group `web_servers`.

Once done your inventory file should look similar to this:

    elvis-1  ansible_host=... ...
    elvis-2  ansible_host=... ...

    [db_servers]
    elvis-2

    [web_servers]
    elvis-1


## Task 3: Add the init role

We have two machines now, and there are some tasks we want to execute on both of them before the
other configuration starts. One of this tasks is APT cache update that you previously did by adding
an `update_cache: yes` to your `apt` module tasks.

Let's add another role named `init` that will be assigned to every machine as the very first step
in out infrastructure deployment. List of tasks in this role should only contain one task so far --
APT cache update (more tasks will be added later). Let's write it this way:

    - name: Update APT cache
      ansible.builtin.apt:
        cache_valid_time: 86400

In this form it will update the APT cache on the first run, and won't do any further updates within
the next 24 hours.

If you do something like this instead:

    - name: Update APT cache  # WRONG! DON'T DO IT!
      ansible.builtin.apt:
        update_cache: yes

Ansible will update the APT cache on every run, and generate the unnecessary change. And as you
remember, changes in the infrastructure should only happen when needed, and the final Ansible run
should produce **no** changes.

Update the `infra.yaml` playbook and add another play (as the very first one) named "Init".
This play should apply the `init` role to all hosts in your infrastrucure.
This play will need administrative privileges to execute the tasks (add `become: yes`).

Now with this `init` role you don't need to add `update_cache: yes` to `apt` module tasks anymore.
You can delete these from the other roles.

Also delete the `users` role from the plays. It's not needed there anymore.
But please keep the role code in the `roles` directory.


## Task 4: Install MySQL server

Create an Ansible role named `mysql` that will install and configure MySQL server.

Add Ansible tasks to ensure that MySQL server is installed on a managed host:
1. Ubuntu package `mysql-server` is good enough for this task
2. use Ansible module `apt`. Note that you don't need to set the `update_cache: yes` anymore --
   APT cache is updated in the `init` role now
3. Ensure that the MySQL server is started and enabled to run on system boot (service name: `mysql`)

Add another play named "Database server" to `infra.yaml` playbook. It should apply `mysql` roles to
all machines from `db_servers` group. This play should be added after "Init" but before the
"Web server", so the playbook should look something like this:

    - name: Init
      ...

    - name: Database server
      ...

    - name: Web server
      ...

Run this command to apply the changes:

    ansible-playbook infra.yaml

You can verify that the MySQL service is started by running this command manually on a managed host:

    systemctl status mysql

If you'vbe done everything correctly you should see these two lines in the output:

    Loaded: loaded (/lib/systemd/system/mysql.service; enabled; vendor preset: enabled)
    Active: active (running) since Sun 2021-09-19 15:53:48 UTC; 4min 38s ago

Times will be different of course. If you see something else -- please fix it before moving forward.


## Task 5: Configure MySQL server

By default this MySQL server daemon will bind to local interface only.
This means that only local connections (from the same host) will work.
You can check it by running this command on the managed host (3306 is the default MySQL port):

	$ sudo ss -lnpt | grep 3306
    LISTEN  0  151  127.0.0.1:3306  0.0.0.0:*  users:(("mysqld",pid=9001,fd=24))
                    ^-------^
                      This

This MySQL server behavior is configured in `/etc/mysql/mysql.conf.d/mysqld.cnf` file (if you
installed the package mentioned above) by this setting:

    [mysqld]
    bind-address = 127.0.0.1

This behavior needs to be changed -- web application will connect to the database from the different
host, so MySQL server should bind to public interface to accept external connections. Easiest way to
achive this is to configure MySQL to bind to `0.0.0.0` which means 'any possible public interface
on this host'.

> Most of the tutorials in the Internet will suggest you to change `/etc/mysq/mysql.cnf` or
> `/etc/mysql/mysql.conf.d/mysqld.cnf` or any other similar file. It would work, but there is a
> better way -- you can _override_ the configuration instead of changing it.

Add the `/etc/mysql/mysql.conf.d/override.cnf` file to the managed host with the following content:

    [mysqld]
    bind-address = 0.0.0.0

It will override the existing `[mysqld]:bind-address` setting from the default configuration file --
no need to even touch that file. Awesome!

MySQL server needs to be restarted to apply the change. Use Ansible handlers for that -- you can
find more info about Ansible handlers in the [previous lecture slides](../03-web-app).

Once your role is updated, run Ansible again to apply the changes:

    ansible-playbook infra.yaml

Check the output carefully.
Make sure that MySQL server was actually restarted, and configuration override was applied!

If you have done everything correctly MySQL server should bind to public interface now.
Run this command on a managed host again to verify that:

	$ sudo ss -lnpt | grep 3306
    LISTEN  0  151  0.0.0.0:3306  0.0.0.0:*  users:(("mysqld",pid=9001,fd=24))
                    ^-----^
                      This is what you should see

If you see something else in the output, please fix it before moving forward.


## Task 6: Add MySQL connection variables

Web application will use this MySQL server as a storage backend --
so the application needs its own database, and credentials to access it.

MySQL connection parameters: host, database name, user and password -- are different for different
deployments, and are shared among different roles and tasks. These are clear candidates for
_variables_. Let's define them first.

Create a file named `group_vars/all.yaml` in your Ansible repository and define the variables there:

    mysql_host: 192.168.42.xxx
    mysql_database: agama
    mysql_user: agama
    mysql_password: !vault |
              $ANSIBLE_VAULT;1.1;AES256
              61383032323739633432663361343366396634613831346231303935396264623764306537373030
              3565623834333662626562303533636364366665663630370a613562626463623263633162653634
              62613637353161336437636663393338356437663933623061303438306634616434373837383439
              3361303630633039340a323433646332316634643735613936386131306662346563313535386663
              3132

Internal IP address (`192.168.42.xxx`) of your database server can be found
[on your page](http://193.40.156.86/students.html). Note that this _internal_ address is different
from the one that you have added to the inventory file:
 - you (and Ansible) connect to the _public_ IP (`193.40.156.86`) of the managed host
 - other hosts in the same network connect to _internal_ IP

Plase use the "<yourname>-2" machine address; we defined it as database server in the task 2.

Password **must** be encrypted.
You can get the encrypted value using Ansible Vault you have set up in the task 1:

    ansible-vault encrypt_string <mysql-password-for-agama-here>

Simplest way to test you solution here is to run the Ansible playbook again:

    ansible-playbook infra.yaml

It does not use variables yet, but still reads the variable file. If this file has syntax errors --
Ansible will print an error. If it executed successfully -- your variables file is likely fine.


## Task 7: Add MySQL database

Add another task to `roles/mysql/tasks/main.yaml` to create a MySQL database:
 - add this task after the one that starts and enables the MySQL server; MySQL server should be
   running before you can create databases
 - use Ansible module
   [mysql_db](https://docs.ansible.com/ansible/latest/collections/community/mysql/mysql_db_module.html);
   note the module name: it's community module and it's named `community.mysql.mysql_db`, not
   `ansible.builtin.<something>` as others you've seen before

Use the variables you have just defined:

    - name: MySQL database
      community.mysql.mysql_db:
        name: "{{ mysql_database }}"

Note the quotes around `{{ ... }}`. These are needed, otherwise Ansible will fail to parse the code.

On a first try Ansible may fail with an error saying

    The PyMySQL (Python 2.7 and Python 3.X) or MySQL-python (Python 2.X) module is required.

Ansible needs a Python library on the managed host to connect to MySQL and make required changes.
This library is called PyMySQL and can be installed as `python3-pymysql` package from the Ubuntu APT
repository.

Another error you may see is

    unable to find /root/.my.cnf.
    Exception message: (1698, "Access denied for user 'root'@'localhost'")

This means that Ansible could not authorize in the MySQL to make the required changes.

Don't hurry to create this file though. MySQL server (if installed from the package mentioned above)
is configured to authorize local `root` user already. This is done via local UNIX socket file, all
you need to do is to instruct Ansible how to use it. Try this instead:

    - name: MySQL database
      community.mysql.mysql_db:
        name: "{{ mysql_database }}"
        login_unix_socket: /var/run/mysqld/mysqld.sock

Exact socket file location can be found in the MySQL configuration file:
`/etc/mysql/mysql.conf.d/mysqld.cnf`.

Once done, run Ansible to apply the changes:

    ansible-playbook infra.yaml

If everything is done correctly, database `agama` should be created in the MySQL.
You can check that by running this command on a managed host:

    sudo mysql -e "SHOW TABLES" agama

IF database exists (good) it will produce no output. Otherwise an error will be printed:

    ERROR 1049 (42000): Unknown database 'agama'

If you get this error, please fix it before moving forward.


## Task 8: Add MySQL user

Add another task to `mysql` role to create a MySQL user for the web application:
 - use Ansible module
[mysql_user](https://docs.ansible.com/ansible/latest/collections/community/mysql/mysql_user_module.html)
 - use `login_unix_socket` trick from the previous task to get rid of "Access denied" error

Start with this:

    - name: MySQL user
      community.mysql.mysql_user:
        name: "{{ mysql_user }}"
        password: "{{ mysql_password }}"

By default Ansible will create a MySQL user that will only be able to login from the same machine
(called `agama@localhost`). We need a remote user that can login from a different host, it would be
called `agama@%` in MySQL where `%` means 'any host'.
This can be configured using `host` attribute of the `mysql_user` module:

    host: "%"

While creating the MySQL user for your application, make sure that it has access **only** to its own
database (not the other databases). This can be achived with `priv` attribute of the `mysql_user`
module, and `mysql_database` variable you defined in the task 6:

    priv: "{{ mysql_database }}.*:ALL"

Once ready, run the Ansible to apply the changes:

    ansible-playbook infra.yaml

After MySQL database and user are created you can verify that this user can login by running this
command (manually) on the database server (assuming user name is `agama`):

	mysql -u agama -p

It will ask you for the password you encrypted previously (`mysql_password` variable) and once
authorized you should get into MySQL console:

	mysql>

If that works, your MySQL server is set up correctly.

Type `exit` to quit the MySQL console.

If something doesn't work here -- please fix it before moving forward.


## Task 9: Reconfigure AGAMA to use MySQL

Finally, it's time to configure our web application to use MySQL as the storage backend.
This is an easy task now :)

Roles from the previous lab have almost everything needed already. We just need a few minor tweaks.

In the `uwsgi` role:
1. move `files/agama.ini` file to `templates/agama.ini`
2. update the task that uploads the AGAMA app configuration to use Ansible module
   [template](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html
   instead of `copy`
3. update the `agama.ini` template and replace the `AGAMA_DATABASE_URI` value to use MySQL server
   you have set up in the previous tasks instead of SQLite file

[AGAMA docs](https://github.com/hudolejev/agama/#running) have the example how to configure MySQL
connection.

Note: when using MySQL backend AGAMA (namely, Python on which it's written) needs additional library
to connect to MySQL. It's already familiar to you `python3-pymysql` from the task 6, but now it also
needs to be installed on the _app server_.

In the `agama` role, update the task that installs AGAMA dependencies to include another package:

    ansible.builtin.apt:
      name:
        - python3-flask-sqlalchemy
        - python3-pymysql           <-- add this


Once ready, run the Ansible playbook to apply changes:

    ansible-playbook infra.yaml

Make sure that uWSGI is restarted after AGAMA configuration file is changed.

If you have done everything correctly AGAMA should be served from your web server public interface.
You can ensure that it uses the MySQL backend by doing this:
1. Add some items, or delete the default ones
2. SSH to the MySQL server and run

        sudo mysql -e "SELECT * FROM agama.item" agama

You should see your recent changes there:

    +----+-----------------------------------------------+-------+
    | id | value                                         | state |
    +----+-----------------------------------------------+-------+
    |  1 | A pre-created item with no particular meaning |     1 |
    |  2 | Another even less meaningful item             |     0 |
    |  3 | I HAVE JUST ADDED THIS TO TEST MYSQL BACKEND  |     0 |  <-- here it is
    +----+-----------------------------------------------+-------+

If AGAMA is not working, make sure to check the uWSGI logs. One often problem is AGAMA trying to use the
wrong Python MySQL library. If you see this error in uWSGI log:

	ModuleNotFoundError: No module named 'MySQLdb'

then it's exactly this case. Workaround is to tell AGAMA which exact Python MySQL library to use.
For this, change the `AGAMA_DATABASE_URI` in the uWSGI configuration file template for AGAMA to something like

	AGAMA_DATABASE_URI=mysql+pymysql://...
	                        ^------^
	                        Add this

Run the Asnible again to apply changes, and check if everything is working as expected.

That's it! All done! That was a long lab (:


## Expected result

Your repository contains these files and directories:

	ansible.cfg
	group_vars/all.yaml
	hosts
	infra.yaml
	roles/agama/tasks/main.yaml
	roles/mysql/tasks/main.yaml
	roles/uwsgi/tasks/main.yaml

Your repository also contains all the required files from the previous labs.

Your repository **does not contain** Ansible Vault master password.

Your deployment customizations: MySQL host, database name, user and password -- are variables and
are stored in `group_vars/all.yaml` file.

Web application that uses MySQL backend, and the MySQL server itself are installed and configured,
each on a separate machine, by running this command once:

	ansible-playbook infra.yaml

Running the same command again does not make any changes to any of the managed hosts.

AGAMA web application is available on
[your public URL](http://193.40.156.86/students.html) -- only on this host that you set up as a web
server, and not on the other one.
