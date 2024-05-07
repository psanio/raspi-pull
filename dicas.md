ansible.cfg:
``` xml
[defaults]
inventory = ./hosts
callbacks_enabled = timer, profile_tasks, profile_roles
```
hosts
``` xml
localhost ansible_connection=local ansible_python_interpreter="/usr/bin/env python"
```
2. Disable fact gathering
When a playbook executes, each play runs a hidden task, called gathering facts, using the setup module. This gathers information about the remote node you're automating, and the details are available under the variable ansible_facts. But if you're not using these details in your playbook anywhere, then this is a waste of time. You can disable this operation by setting gather_facts: False in the play.

4. Configure SSH optimization
Establishing a secure shell (SSH) connection is a relatively slow process that runs in the background. The global execution time increases significantly when you have more tasks in a playbook and more managed nodes to execute the tasks.

You can use ControlMaster and ControlPersist features in ansible.cfg (in the ssh_connection section) to mitigate this issue.

ControlMaster allows multiple simultaneous SSH sessions with a remote host to use a single network connection. This saves time on an SSH connection's initial processes because later SSH sessions use the first SSH connection for task execution.
ControlPersist indicates how long the SSH keeps an idle connection open in the background. For example, ControlPersist=60s keeps the connection idle for 60 seconds:
[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
---
But the better and more efficient method is this:

- name: Install httpd and firewalld
  ansible.builtin.yum:
    name: 
      - httpd
      - firewalld
      - git
    state: latest
---
2. Avoid copy loops and use the synchronize module
When you have multiple files to copy into the same directory, synchronize modules rather than using multiple copy modules or loops:

- name: Copy application data
  synchronize:
    src: app_data/
    dest: /opt/web_app/data
---
4. Make configuration templates
You might use multiple lineinfile and blockinfile tasks to manage and configure a single file. This approach creates a very long playbook. And when there's configuration drift, you must edit this lineinfile task with a different regex. However, you can use a Jinja2 template to create any level of complex files and use the template module (or filter) to configure managed nodes.

For instance, you can copy a complex Nginx web server configuration using the template module with a Jinja2 template.

Your Jinja2 template: nginxd.conf.j2

# nginx configuration for wp-test
server {
    root /var/www/{{ website_root_dir }};
    index index.php index.html index.htm;
    server_name {{ website_name }}.com www.{{ website_name }}.com;
    access_log /var/log/nginx/access_{{ website_name }}-com.log;
    error_log /var/log/nginx/error_{{ website_name }}-com.log;
...<output removed>...
    ssl_certificate /etc/letsencrypt/live/{{ website_name }}.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/{{ website_name }}.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}
server {
    if ($host = www.{{ website_name }}.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot
...<output removed>...
    server_name {{ website_name }}.com www.{{ website_name }}.com;
    return 404; # managed by Certbot
}
The template module replaces the variable (the code {{ website_root_dir }}) with values provided in your playbook:

---
- name: Configure the nginx Web Server
  hosts: web_servers
  become: True 
  vars:
    website_name: myawesomeblog
    website_root_dir: /var/www/myawesomeblogdata
  tasks:
    - name: Copy nginx configuration
      template:
        src: nginxd.conf.j2
        dest: /etc/nginx/sites-enabled/{{ website_name }}.conf

---
Testing
Rigorously test playbooks locally before having nodes pull configurations:
``` Shell
ansible-playbook --syntax-check site.yml 
ansible-playbook --list-tasks site.yml
```
Logging
Direct Ansible pull command output to a log file for troubleshooting:
``` Shell
ansible-pull site.yml > /var/log/ansible-pull.log 2>&1
```
Security
Use SSH keys instead of passwords when pulling Git repositories containing sensitive Ansible code:
``` Shell
ansible-pull -U git@example.com/ansible-configs.git --private-key=/home/user/.ssh/id_rsa
```
Scheduling Regular Pulls with Cron
While running ansible-pull manually works, we want configurations to be autonomously applied frequently.

Cron is perfect for having nodes periodically fetching for updates.

A sample cron entry would be:
``` Crontab
*/30 * * * * ansible-pull -U git@example.com/ansible-configs.git httpd.yml > /var/log/ansible-pull.log
```
