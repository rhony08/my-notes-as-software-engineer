# Ansible for Configuration Management

You've SSH'd into 20 servers to update a config file. You've run the same `apt-get` command five times because you keep forgetting which servers you already did. You've had production incidents because one machine had a slightly different Nginx config than the others.

Manual server management works—until you have more than a handful of machines. Then it breaks. Hard.

**Ansible** is the tool that makes this stop. It's an agentless configuration management tool by Red Hat that lets you define *what* your servers should look like, and it makes them match that definition. No agents to install, no complex PKI setup—just SSH.

## What Makes Ansible Different

Most config management tools (Puppet, Chef, Salt) need an agent installed on every managed machine. Ansible says nah—it uses SSH and Python, both of which are already on practically every Linux server.

| Feature | Ansible | Puppet/Chef |
|---------|---------|-------------|
| Agent required? | No (SSH only) | Yes, on every node |
| Learning curve | Low (YAML) | Medium-high (DSL/ruby) |
| Push vs Pull | Push mode | Pull mode (agent polls master) |
| Setup time | Minutes | Hours |
| State management | Declarative & imperative | Mostly declarative |

**The trade-off:** Agentless means Ansible runs commands over SSH one by one. For 5 servers, this is instant. For 500, it gets slow. Puppet's agent model scales better because agents run in parallel.

## Your First Ansible Project

Ansible organizes work into **playbooks** (YAML files) and **inventory** (what servers to target).

### Inventory: Who Are We Managing?

```ini
# inventory.ini
[webservers]
web01.example.com
web02.example.com

[databases]
db01.example.com ansible_user=deploy

[all:vars]
ansible_user=ubuntu
ansible_python_interpreter=/usr/bin/python3
```

You can define groups (`[webservers]`), individual variables, and shared default variables. Simple.

### A Playbook: Define the State

```yaml
# playbooks/nginx-setup.yml
---
- name: Configure Nginx on all web servers
  hosts: webservers
  become: yes  # use sudo

  tasks:
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present  # "present" = install if missing
        update_cache: yes

    - name: Copy Nginx config
      ansible.builtin.template:
        src: ../templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
      notify: restart nginx  # only restart if config changed

    - name: Ensure Nginx is running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes  # start on boot

  handlers:
    - name: restart nginx
      ansible.builtin.service:
        name: nginx
        state: restarted
```

This playbook says:
- On every `webservers` machine
- Install Nginx if missing
- Copy a config template (from `templates/nginx.conf.j2`)
- Restart Nginx only if the config changed (that's what `notify` + `handlers` does)
- Make sure Nginx is running and will start on boot

**That's idempotent.** Run it ten times. The first time, it installs Nginx. The next nine, it does nothing because everything's already in the right state.

## ❌ vs ✅: Manual vs Ansible

**❌ Manual approach (what you're doing now):**

```bash
ssh web01
# apt-get update
# apt-get install nginx
# vim /etc/nginx/nginx.conf
# systemctl restart nginx
# ... now manually repeat on web02, web03, etc.
```

Result: Inconsistent configs, forgotten servers, "I swear I did that one."

**✅ Ansible approach:**

```bash
ansible-playbook -i inventory.ini playbooks/nginx-setup.yml
```

Result: All servers identical. Config changes are version-controlled. New server joins the inventory and gets configured automatically.

## Beyond Basics: Ansible Modules

Ansible has hundreds of built-in modules for everything:

```yaml
- name: Create deployment user
  ansible.builtin.user:
    name: deploy
    state: present
    groups: sudo,www-data
    create_home: yes

- name: Clone application repo
  ansible.builtin.git:
    repo: "https://github.com/myorg/myapp.git"
    dest: /var/www/myapp
    version: main
    force: yes  # overwrite local changes

- name: Set env variable in .env file
  ansible.builtin.lineinfile:
    path: /opt/myapp/.env
    regexp: '^DATABASE_URL='
    line: 'DATABASE_URL=postgres://db01:5432/myapp'
```

`lineinfile` is particularly handy—find a matching line in a file and replace it. Great for config tweaks without overwriting entire files.

## Variables and Templates

Hardcoding values in playbooks defeats the purpose. Use **variables** and **Jinja2 templates** instead.

```yaml
# playbooks/app-deploy.yml
---
- name: Deploy application
  hosts: all
  vars:
    app_port: 3000
    node_env: "{{ env | default('staging') }}"

  tasks:
    - name: Render app config
      ansible.builtin.template:
        src: app.env.j2
        dest: /opt/myapp/.env
```

Then in your template `app.env.j2`:

```jinja2
PORT={{ app_port }}
NODE_ENV={{ node_env }}
DATABASE_URL={{ db_url }}
REDIS_HOST={{ redis_host }}
```

Running with different environments:

```bash
# Staging (default)
ansible-playbook -i staging.ini app-deploy.yml

# Production with override
ansible-playbook -i production.ini app-deploy.yml -e "env=production"
```

Variables cascade: inventory vars > playbook vars > extra vars (`-e`). The more specific wins.

## Idempotency: The Core Insight

Ansible's superpower isn't running commands—it's **ensuring state**. Each module checks the current state before making changes.

Take the `apt` module:
- `state: present` → "Is Nginx installed? If yes, skip. If no, install."
- `state: latest` → "Is Nginx at the latest version? If yes, skip. If no, upgrade."

This means:
- ✅ Run on a fresh server → installs everything
- ✅ Run again on the same server → does nothing (idempotent!)
- ✅ Run after someone manually messed with config → Ansible fixes it
- ✅ Run after a security patch → only the affected services restart

**Why it matters:** You can schedule Ansible to run every 15 minutes as drift correction. Any manual change gets reverted automatically. Chaos theory in reverse.

## Roles: Organizing Playbooks

As your playbooks grow, you need structure. **Roles** are Ansible's answer:

```
ansible-project/
├── inventory.ini
├── site.yml              # master playbook
├── roles/
│   ├── nginx/
│   │   ├── tasks/main.yml
│   │   ├── templates/nginx.conf.j2
│   │   ├── handlers/main.yml
│   │   └── vars/main.yml
│   ├── postgres/
│   │   ├── tasks/main.yml
│   │   ├── templates/postgresql.conf.j2
│   │   └── vars/main.yml
│   └── app/
│       ├── tasks/main.yml
│       └── vars/main.yml
```

Your master playbook becomes:

```yaml
# site.yml
---
- name: All-in-one setup
  hosts: all
  roles:
    - common
    - nginx
    - app
    - postgres
```

Each role is self-contained. Share them across projects. Use **Ansible Galaxy** to pull community roles (common patterns like `geerlingguy.nginx`).

## When NOT to Use Ansible

Ansible isn't the right tool for everything:

- **Dynamic scaling** — If you're constantly spinning up/down 1000 instances, Ansible's sequential SSH model gets slow. Consider Terraform + Packer + immutable images.
- **Windows-heavy environments** — Ansible can manage Windows (WinRM instead of SSH), but it's clunky compared to native tools.
- **Real-time config changes** — Ansible is push-based. For runtime secrets rotation or rapid config changes, something like Consul or etcd is better.
- **Kubernetes** — For container orchestration, use Helm or Kustomize. Ansible can *provision* clusters but shouldn't manage individual pods.

## Common Pitfalls

**1. Forgetting `become: yes`**

```yaml
# This will fail — can't install packages without sudo
- name: Install Nginx
  apt:
    name: nginx
    state: present
```

Always use `become: yes` for system-level tasks.

**2. Wrong Python interpreter on target**

Some distros ship Python 2, some Python 3. Set it explicitly:

```ini
ansible_python_interpreter=/usr/bin/python3
```

**3. Forgetting handlers only fire on changes**

Handlers run *after* all tasks, but only if notified. This is usually what you want, but it means:

```yaml
- name: apt update
  apt:
    update_cache: yes
  changed_when: false  # Don't trigger handlers
```

Use `changed_when: false` for tasks that shouldn't count as "changes."

## Quick Reference: Essential Commands

```bash
# Check connectivity (no changes)
ansible -i inventory.ini all -m ping

# Run ad-hoc command
ansible -i inventory.ini webservers -a "uptime"

# Dry-run a playbook
ansible-playbook -i inventory.ini playbooks/site.yml --check

# Run with verbose output
ansible-playbook -i inventory.ini playbooks/site.yml -vvv

# Run specific tags only
ansible-playbook -i inventory.ini playbooks/site.yml --tags "nginx,config"

# Gather facts about a server
ansible -i inventory.ini web01 -m setup
```

## Takeaways

- **Agentless** is Ansible's killer feature — SSH and Python are all you need, zero setup on target machines
- **Idempotency** means you describe the desired state, not the steps to get there — run it once or 100 times, same result
- **Roles** keep playbooks organized as your infrastructure grows beyond a few files
- Ansible is great for config management, but consider Terraform + immutable images if you're doing heavy dynamic scaling
- `--check` (dry run) is your friend — always test before applying to prod
