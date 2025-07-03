# Introduction to Ansible

Ansible is an IT automation tool to provision, and management of your infrastrcture and softwares. It's much easier than writing scripts for your needs. It uses YAML format to define tasks. You can find comprehensive documentation [here](https://docs.ansible.com).

## Ansible Configuration Files

when you install ansible on your setup, it creates default configuration file under `/etc/ansible/ansible.cfg`. Configuration file is divided into multiple sections, such as defaults, inventory, ssh_connection etc. On each section, there are multiple number of options and values.

Configuration files can be changed for each playground depending on needs. To do this operation, it is necessary to copy configuration file to location where each playbook resides (such as `/opt/web-playoboks/ansible.cfg`). And make necessary changes on those files. By doing this, you can override default configuration files of ansible for given playbook.

Another approach is to create a config file for multiple playbooks and expose this file before running your command. To give an example, assume that we've created a config file under `/opt/ansible-web.cfg`. On any playbook that we want to use this configuration file, we can simply run `$ANSIBLE_CONFIG=/opt/ansible-web.cfg ansible-playbook <name-of-playbook>`. By doing this, we manipulate env variable for ansible configuration and make playbook run for our configration file.

On those config file, you do not have to have all options that ansible needs. Instead, you can simply define parameters that you want to override. For the rest of the parameters, they can be fetched from default configuration files. So it is important to mention order of config files that ansible reads. It basically follows that order:


	1- Config file that we use by manipulating env path.
	2- Config file that resides same path with playbook.
	3- Config file under .ansible.cfg path for each user.
	4- Config file under /etc/ansible

If we need to change only one field on configuration file, it can be done on CLI without creating a new config file. For example:
`ANSIBLE_GATHERING=explicit ansible-playbook <name-of-playbook>` command just overrides gathering options on the config.

`ansible-config list` command shows all possible configration options that can be useful, similarly `ansible-config view` command shows the current config file. `ansible-config dump` command shows the current settings.

## Ansible Inventory

Ansible can work single or multiple infrastuctures at same time. Ansible is agentless, you do not need to install anything on nodes. All we need is SSH (or winrm for windows).

Information about target systems are stored in an **inventory** file. Default inventory file is located at `/etc/ansible/hosts` file, if we do not create any inventory file for a playbook, Ansible will use this default file.

Inventory file is like INI format file, it simply have number of servers listed, one after the other. An example should look like:

```INI
server1.company.com
server2.company.com

[mail]
server3.company.com
server4.company.com

[db]
server5.company.com
server6.company.com
```


It's also possible to give alias to those servers such as:

```INI
server1.company.com
server2.company.com

[mail]
mail1 ansible_host=server3.company.com
mail2 ansible_host=server4.company.com

[db]
server5.company.com
db2 ansible_host= server6.company.com
```

There are different parameters on inventory file that we can use:

 - `ansible_connection` - ssh/winrm/localhost
 - `ansible_port` - 22/5986
 - `ansible_user` - root/administrator
 - `ansible_ssh_pass` - Password (ssh-password-for-linux)



```INI
# Sample Inventory File

# Web Servers
web1 ansible_host=server1.company.com ansible_connection=ssh ansible_user=root ansible_ssh_pass=Password123!
web2 ansible_host=server2.company.com ansible_connection=ssh ansible_user=root ansible_ssh_pass=Password123!
web3 ansible_host=server3.company.com ansible_connection=ssh ansible_user=root ansible_ssh_pass=Password123!
# Database Servers
db1 ansible_host=server4.company.com ansible_connection=winrm ansible_user=administrator ansible_password=Password123!


[web_servers]
web1
web2
web3

[db_servers]
db1

[all_servers:children]
web_servers
db_servers
```

> It is possible to have inventory files in different formats such as YAML.

An example YAML format:

```yaml
all:
	children:
		webservers:
			hosts:
				web1.example.com
				web2.example.com
		dbservers:
			hosts:
				db1.example.com
				db2.example.com
```

As we've described above, it is possible to group and give children to those groups. As an example (for both ini and yaml formats):

```ini
[webservers:children]
webservers_us
webservers_eu

[webservers_us]
server1_us.com ansible_host=192.168.8.101
server2_us.com ansible_host=192.168.8.102

[webservers_eu]
server1_eu.com ansible_host=192.168.0.101
server2_eu.com ansible_host=192.168.0.102
```

```yaml
all:
	children:
		webservers:
			children:
				webservers_us:
					hosts:
						server_1.us.com:
							ansible_host: 192.168.8.101
						server_2.us.com:
							ansible_host: 192.168.8.102
				webservers_eu:
					hosts:
						server1_eu.com:
							ansible_host: 192.168.0.101
						server2_eu.com:
							ansible_host: 192.168.0.102
```

To run ansible-playbook with given inventory file:

```bash
ansible-playbook -i <name-of-inventory-file> app_install.yaml 
```

## Ansible Variables

Variables are used to store information for various use-cases. `ansible_host`, `ansible_connection` that we've used for inventory files are variables for ansible. It is possible to define variables in a different file, or just define variable in any yaml file such as playbooks. As an example of defining and using variable is given below:

```yaml
name: Add DNS server to resolv.conf
hosts: localhost
vars:
	dns_server: 10.1.25.10
tasks:
	- lineinfile:
		path: /etc/resolv.conf
		line: 'nameserver {{ dns_server }}'

```

> Variable definition by curly brackets are called *Jinja2 Templating*.

It is possible to define variables on inventory files aswell.

```ini
web1 dns_server=10.1.25.10 ansible_host=192.168.0.12
```

### Variable Types

#### String Variables: 
Sequences of characters. They can be defined in a playbook, inventory, or assed as command line arguments.

#### Number Variables: 
Integer or floating-point values.

#### Boolean Variables:
Either true or false.

Valid truthy values:

	-	True, 'true', 't', 'yes', 'y', 'on', '1', 1, 1.0

Valid falsy values:

	-	False, 'false', 'f', 'no', 'n', 'off', '0', 0, 0.0

#### List Variables

To hold ordered collection of arrays.

```yaml
packages:
	- nginx
	- postgresql
	- git
```

To use first item on the list, use `{{ packages[0] }}`, to use them all use `{{ packages }}`.

#### Dictionary Variables

To store key-value pairs.

```yaml
user:
	- name: 'abd'
	- surname: 'qwe'
```

To use them, expressions like `{{ user.name }}` should be used.

### Variable Precedence

On ansible, there are different type of variables (group vars, host vars, play vars, and extra vars). Those variables are different precendences. Host variables always overcome group variables, and play variables always overcome host variables. Follow the next example:

```ini
web1 ansible_host= 192.168.0.1
web2 ansible_host= 192.168.0.2 dns_server=10.5.5.4

[web_servers]
web1
web2

[web_servers:vars]
dns_server=10.5.5.3
```

In the xample above, web1 will have dns_server value of 10.5.5.3, whereas web2 will have 10.5.5.4 because host vars always have higher precedence than group vars.

Extra vars are the variables that are defined by CLI.

```bash
ansible-playbook <name-of-playbook> --extra-vars "dns_server=10.5.5.6"
```

Comphrehensive variable orders are given [here](https://gist.github.com/ekreutz/301c3d38a50abbaad38e638d8361a89e).

### Register Output

On a sequenced operation, it is possible to use output of former operation on next ones. This is called register output. I'll give an example to show it.

```yaml
- name: Check /etc/hosts file
  hosts: all
  tasks:
	- shell: cat /etc/hosts
	  register: result
	- debug:
		var: result
```


In the example avove, result of `shell` operation will be inout for `debug` operation as we feed it with a `var`. To check the outputs, and investigate more for your playbook, it is possible to use `-v` flag. (verbose mode):

```bash
ansible-playbook -i inventory <name-of-playbook> -v
```

### Magic Variables

Ansible has its own variable scopes. A variable defined on host level can not be accessed by other hosts as expected. Follow the inventory file below:

```
web1 ansible_host=172.20.1.100
web2 ansible_host=172.20.1.101 dns_server=10.5.5.4
web3 ansible_host=172.20.1.102
```

`dns_server` variable will only be accessible for `web2` host as it is defined for that host level. So for following playbook, only `web2` will be able to get variable.

```yaml
---
- name: Print DNS server
  hosts: all
  tasks:
  	- debug:
  		msg: `{{ dns_server }}`
```

There is a term in Ansible called as `magic variables` that allows us to use host variables (or any host related information - this is called as `hostvars`) for other hosts. By using them, it is actually possible to reach a host variable, or host information from other hosts. Usage of magic variables are given below:

```yaml
---
- name: Print DNS server
  hosts: all
  tasks:
  	- debug:
  		msg: `{{ hostvars['web2'].dns_server }}`

```

It is also possible to reach more information about the hosts with same approach (they are called ansible facts). Examples are:

```bash
`{{ hostvars['web2'].ansible_facts.architecture }}`
`{{ hostvars['web2'].ansible_facts.devices }}`
`{{ hostvars['web2'].ansible_facts.mounts }}`
`{{ hostvars['web2'].ansible_facts.processor }}`


## It is also possible to write same expression given below:

`{{ hostvars['web2']['ansible_facts']['processor'] }}`
```

Similar to `hostvars`, there are much more magic variables, such as `groups`, `group_names`, `inventory_hostname` etc. You can reach comprehensive list [here](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html#information-about-ansible-magic-variables).

### Ansible Facts

General information about the machine that is gathered when Ansible connects a machine. Facts are gathered thanks to `setup` module on Ansible. This module gets installed automatically when ansible is installed.

All facts that are generated are stored variable named `ansible_facts`.

```yaml
---
- name: get facts
  hosts: all
  tasks:
  	- debug:
  		var: ansible_facts
```

> To disable gathering facts, it is possible to do. All we need to do is add a new flag called ass `gather_facts: no` on our playbook. Or similarly it can be disabled on our ansible config file by changing value of flag `gathering` (on default it is set to *implicit* - it can be disabled by setting this value to *explicit*).

## Playbooks

It is where we define what we want do to with ansible. Main logic part of Ansible. All playbooks are written in YAML format. In playbook we define a set of activities to be run on host(s). An example playbook looks like:

```yaml
-
 name: Play 1
 hosts: localhost
 tasks:
 	- name: Execute command 'date'
 	  command: date

 	- name: Execute script on server
 	  script: test_script.sh

 	- name: Install httpd service
 	  yum:
 	  	name: httpd
 	  	state: present

 	 - name: start web server
 	   service:
 	   	name: httpd
 	   	state: started
```

> Pay attention that tasks are ordered list, so actions listed in this field will executed sequentially.

> When a group is specified as hosts, all actions will be performed on hosts *simultaneously*.

On each task, we may have different options as we can see from the playbook above. As an example, we have `command`, `script`, `yum` and `service`. Each of those fields are called **modules**. There are hundreds of other modules available on Ansible. To get more information about the modules, it is possible to run `ansible-doc -l` command.

To run playbook, executing is easy: `ansible-playbook <name-of-playbook>`

### Verification of Playbooks

Before using playbooks, it is important to verify playbook to prevent unnecessary failures and shutdowns. It is a crucial practice (rehersal before real action). There are two modes that we use to verify a playbook in Ansible. There are:

- **Check Mode:** Check mode is a dry run mode where no actual changes are made on hosts. Check mode allows preview of playbook changes without applying them. To use it use `--check` option. No all Ansible mode supports check mode, modules that does not support check mode will be skipped during this process.
- **Diff Mode:** Diff mode provides a before-and-after comparison of playbook changes. To run this mode, use `--diff` option.
- **Syntax Check Mode:** To ensure playbook syntax is error-free. We can use it by `--syntax-check` option.

> Ansible-lint is a cool feature that lints your playbooks to make easier to read and maintain. It provides a standartization for your playbooks. It is a CLI and checks playbook in a wide aspects. To use ansible-lint, simply call it like we do for playbook. For example: `ansible-lint <name-of-playbook>`. No output means there is nothing to worry about.

### Conditionals

It's possible to create conditional playbooks depending on needs. Condition statement is declared with `when` flag under task. Condition should be any check that depends on our needs. It's possible to use `and` or `or` operator on conditions. As an example:

```yaml
---
- name: Install NGINX
	hosts: all
	tasks:
		- name: Install NGINX on Debian
			apt:
				name: nginx
				state: present
			when: ansible_os_family == "Debian" and ansible_distribution_version == "16.04"
		- name: Install NGINX on Redhat
			yum:
				name: nginx
				state: present
			when: ansible_os_family == "RedHat" or ansible_os_family == "SUSE"
```


### Loops

Similar to conditionals, it is possible to create loops on playbook files. To create loop, we use `loop` keyword. An example is given below:

```yaml
---
- name: Install Softwares
	hosts: all
	vars:
		packages:
			- name: nginx
				required: True
			- name: mysql
				required: True
			- name: apache
				required: False
	tasks:
		- name: Install "{{ item.name }}" on Debian
			apt:
				name: "{{ item.name }}"
				state: present
		when: item.required == True
		loop: "{{ packages }}"
```

**Notice** that, when we use loops and give an array to it, on each iteration, we can use elements by using `item` key. To make it clearer, check the example above again.

> Loop directive has recently added to Ansible. Previously we had `with_*` directive. `with_*` is a lookup plugin.

Check the example below with `with_*` plugin:

```yaml
---
- name: Install Softwares
	hosts: all
	tasks:
		- name: Install "{{ item.name }}" on Debian
			apt:
				name: "{{ item.name }}"
				state: present
		when: item.required == True
		with_packages:
			- name: nginx
				required: True
			- name: mysql
				required: True
			- name: apache
				required: False
```

## Ansible Modules

Modules are tools that we can use on playbooks to operate desired operations. They are the key component for ansible. You can do various operations such as system level operations, or command typing etc.

Popular module categories:

	-	System
	- Commands
	- Files
	- Database
	- Cloud
	- Windows

Comprehensive list of modules can be found [here](https://docs.ansible.com/ansible/latest/collections/index_module.html).

## Ansible Plugins

Ansible plugins provide extensibility and customization options beyond the core Ansible features. An Ansible plugin is a piece of code that modifies the funcitonality of Ansible. Plugins provide a flexible and powerful way to tailor Ansible.

	- Dynamic Inventory Plugin
	- Module Plugin
	- Action Plugin
	- Other Plugins (lookup plugins, filter plugins, connection plugins etc.)

Comprehensive list of plugins can be found [here](https://docs.ansible.com/ansible/latest/collections/index_inventory.html).

## Ansible Handlers

Ansible handlers helps us to automate repeated tasks by given instructions. Handlers run task(s) whenever we want to trigger them. In Ansible:

	-	Tasks triggered by events/notifications.
	- Handlers defined in playbooks, and they are executed when notified by a task
	- They help us to manage actions based on system state/configuration changes

As an example given below, we can see that, `notify` directive works as a trigger to make specific handler run after a given task. On this playbook, whenever we copy our application code, "Restart Application Service" handler will be executed after "Copy Application Code" task is done.

```yml
- name: Deploy Application
	hosts: application_servers
	tasks:
		- name: Copy Application Code
			copy:
				src: app_code/
				dest: /opt/application
			notify: Restart Application Service
	handlers:
		- name: Restart Application Service
			service:
				name: application_service
				state: restarted
```

**Important Note:** The key point here is understanding how handlers work in Ansible. When multiple tasks notify the same handler, the handler will only run once at the end of the playbook, regardless of how many tasks notify it.

```yaml
- name: Test Handler Execution
  hosts: localhost
  tasks:
    - name: Copy file1.conf
      copy:
        src: files/file1.conf
        dest: /tmp/file1.conf
      notify: Sample Handler

    - name: Copy file2.conf
      copy:
        src: files/file2.conf
        dest: /tmp/file2.conf
      notify: Sample Handler

  handlers:
    - name: Sample Handler
      debug:
        msg: "Handler has been triggered!"
```

Handler will be exetuced single time only when all tasks are done!

## Roles in Ansible

Roles let you automatically load related vars, files, tasks, handlers, and other Ansible artifacts based on a known file structure. After you group your content into roles, you can easily reuse them and share them with other users.

To make it clearer, we can go through of an example. Assume that you have a database server and you make configuration for them. For future, we may need to add a new server to make database run on multiple servers. In such a case, it would be a good idea to define database initialization as a role, and use them when necessary.

```yml
tasks:
	- name: Install Pre-Requistes
		yum: name=pre-req-packages state=present
	- name: Install MySQL packages
		yum: name=mysql state=present
	- name: Start MySQL service
		yum: name=mysql state=started
	- name: configure database
		mysql_db: name=db1 state=present

```

Then, we can use this role in our playbooks (for other projects or same project, does not matter):

```yml
- name: Install and Configure MySQL
	hosts: db-servers
	roles:
		- mysql

```

Roles helps to organize our operations. It is also a good platform to share your works for other Ansible users. For sharing and downloading roles, [galaxy](https://galaxy.ansible.com/ui/) is used. Ansible Galaxy is also a great tool to create roles in a structured way. By using `ansible-galaxy init <role-name>` helps us.

Roles can be defined on host level or playbook level. Generally we would have a `roles` folder that contains information like that:

```
roles/
    common/               # this hierarchy represents a "role"
        tasks/            #
            main.yml      #  <-- tasks file can include smaller files if warranted
        handlers/         #
            main.yml      #  <-- handlers file
        templates/        #  <-- files for use with the template resource
            ntp.conf.j2   #  <------- templates end in .j2
        files/            #
            bar.txt       #  <-- files for use with the copy resource
            foo.sh        #  <-- script files for use with the script resource
        vars/             #
            main.yml      #  <-- variables associated with this role
        defaults/         #
            main.yml      #  <-- default lower priority variables for this role
        meta/             #
            main.yml      #  <-- role dependencies
        library/          # roles can also include custom modules
        module_utils/     # roles can also include custom module_utils
        lookup_plugins/   # or other types of plugins, like lookup in this case

    webtier/              # same kind of structure as "common" was above, done for the webtier role
    monitoring/           # ""
    fooapp/               # ""
```

Within our playbook, we can either move role folder to same directory that our playbook resides, or define path of role in our playbook. Roles folder stays in `/etc/ansible/roles` directory by default. 

There are some useful commands with ansible galaxy:

- `ansible-galaxy search <rolename>` to find related role
- `ansible-galaxy install <rolename>` to install a role (role is extracted in default role directory)

## Ansible Collections

Collections are bundles of Ansible content: they can include modules, plugins, roles, playbooks, and documentation. A collection is a structured distribution format, usually focused on a specific domain, vendor, or purpose. Benefits of collections:

-	Expanded functionality
-	Modularity and Reuseability
-	Simplified Distribution and Management

```yaml
- name: Install nginx
  ansible.builtin.package:
    name: nginx
    state: present

```

Here:

- ansible.builtin.package is a module
- It might be part of the ansible.builtin collection


To install a collection, we can use `ansible-galaxy collection install <name-of collection>` command. For a playbook that specifies collections in it, we can basically install them all by using `ansible-galaxy collection install -r <name-of-playbook>`.

## Templating Tips

As you remember, we use `Jinja` templating tool on Ansible. Below, you can find some tips related to templating (assume my_name value has value of *Bond*):

- The name is {{ my_name }} => The name is Bond
- The name is {{ my_name | upper }} => The name is BOND
- The name is {{ my_name | lower }} => The name is bond
- The name is {{ my_name | replace("Bond", "Bourne") }} => The name is Bourne
- The name is {{ first_name | default("James") }} {{ my_name }} => The name is James Bond
- {{ [1,2,3] | min }} => 1 (similarly we have `max, unique, union, intersect, random, join` methods)

Loops in jinja: 
```jinja
{% for number in [0, 1,2,3,4] %}
	{{ number }}
{% endfor %}
```

Conditionals in jinja:
```jinja
{% for number in [0, 1,2,3,4] %}
	{% if number == 2 %}
		{{ number }}
	{% endif %}
{% endfor %}
```
## Useful Commands - Tips

- ansible -i inventory node01 -m ping -v
- ansible -i inventory all -m ping 
- ansible-playbook -i localhost, command.yml -vv
