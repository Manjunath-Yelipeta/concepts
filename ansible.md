# Ansible

## Why Not Just Shell Scripts?

Shell scripts work, but they fall apart when you're managing many servers:

- You SSH into each server and run the script manually — doesn't scale
- Different servers might have different OS versions, users, paths — scripts break
- No easy way to run the same script on 50 servers at once
- Scripts are not idempotent by default — you have to write that logic yourself
- Hard to track what state each server is in

This is where **configuration management** tools like Ansible come in.

---

## What is Configuration Management?

When you spin up a fresh server, it's just a blank machine. You have to install packages, create users, update configs, start services — that's **configuring the server**.

**Configuration management** means doing all of this — create, read, update, delete — through code/scripts, not manually. So the server's state is always predictable and reproducible.

> In shell: everything is a command.  
> In Ansible: everything is a **module** (a pre-built, reusable unit of work).

---

## Push vs Pull Architecture

Most config management tools work in one of two ways:

**Pull** — every server checks in with a central server periodically asking "do I have new configs?" (like you going to the courier office every day to check if your package arrived)
- Wastes resources
- Unnecessary traffic even when nothing changed
- Examples: Chef, Puppet

**Push** — you sit at your machine and push config to all servers when you need to (like the courier delivering to your door when there's something to deliver)
- You control when changes happen
- No agent needed on target servers
- Ansible works this way — over plain SSH

---

## Inventory File

Ansible needs to know which servers to talk to. That list is called an **inventory**.

Simplest form — just an IP or hostname:

```
172.31.27.248
```

You can group servers:

```ini
[frontend]
172.31.10.5
172.31.10.6

[backend]
172.31.20.5

[db]
172.31.30.5
```

Then you can target a group by name in your commands or playbooks.

---

## Ad-hoc Commands

One-off commands without writing a playbook. Good for quick tasks or testing connectivity.

```bash
ansible all -i 172.31.27.248, -e ansible_user=ec2-user -e ansible_password=DevOps321 -m ping
```

| Part | What it means |
|------|---------------|
| `all` | target all hosts in inventory |
| `-i 172.31.27.248,` | inline inventory (comma at end is required for single IP) |
| `-e ansible_user=...` | extra variable — which user to SSH as |
| `-e ansible_password=...` | SSH password |
| `-m ping` | module to run (`ping` checks if Ansible can connect) |

The `ping` module doesn't ping like ICMP ping — it checks if Ansible can SSH in and Python is available on the target.

---

## Playbooks

When you have multiple tasks to run against a server, you write them in a file — that's a **playbook**.

```yaml
- name: ping the server
  hosts: frontend
  tasks:
    - name: ping the server
      ping:
```

- `hosts` — which group from your inventory to target
- `tasks` — list of modules to run in order
- Each task has a `name` (for readability) and a module (`ping`, `dnf`, `copy`, etc.)

---

## YAML Basics

Playbooks are written in YAML. It's just a structured way to write key-value data — like a form.

```yaml
name: sivakumar
branch: hyderabad
date: 01-jun-2026
```

**List** — use `-`:

```yaml
tasks:
  - name: install nginx
  - name: start nginx
```

**Map / nested** — indent under a key:

```yaml
hosts:
  frontend:
    ip: 172.31.10.5
```

Key rule: **indentation matters** — use spaces, never tabs.

---

## Variables

Variables let you avoid hardcoding values. You reference them with `{{ variable_name }}`.

### Play-level vars

Defined in the playbook itself, available to all tasks in that play.

```yaml
- name: demo
  hosts: localhost
  connection: local
  vars:
    COURSE: "Ansible"
    TRAINER: "sivakumar"
    DURATION: "150HRS"
  tasks:
  - debug:
      msg: "Learning {{ COURSE }} with {{ TRAINER }} for {{ DURATION }}"
```

### Task-level vars

Defined inside a specific task — overrides play-level vars for that task only.

```yaml
  tasks:
  - name: print info
    vars:
      COURSE: "DevSecOps with AIOps"  # overrides play-level COURSE
    debug:
      msg: "Learning {{ COURSE }}"
```

### vars_files

Keep variables in a separate YAML file and load it in the playbook. Cleaner for large configs.

```yaml
# course.yaml
COURSE: Ansible
TRAINER: sivakumar
DURATION: 10HRS
```

```yaml
- name: demo
  hosts: local
  connection: local
  vars_files:
  - course.yaml
  tasks:
  - debug:
      msg: "{{ COURSE }} by {{ TRAINER }}"
```

### vars_prompt

Ask the user for input at runtime (like `read` in shell).

```yaml
  vars_prompt:
  - name: username
    prompt: "Please enter username"
    private: false       # shows input as you type

  - name: password
    prompt: "Please enter password"
    private: true        # hides input
```

### group_vars

Variables scoped to an inventory group. Ansible auto-loads files from a `group_vars/` folder next to your playbook.

```
group_vars/
  all.yaml        # applies to every host
  frontend.yaml   # applies only to [frontend] group
```

```yaml
# group_vars/frontend.yaml
COURSE: "Ansible from group_vars"
TRAINER: "Sivakumar"
```

### Command-line vars (-e)

Pass variables at runtime — highest priority, overrides everything.

```bash
ansible-playbook 09-vars.yaml -e name=Sivakumar
```

```yaml
  tasks:
  - name: print name
    debug:
      msg: "Hello {{ name }}"
```

### Variable Preference Order (highest → lowest)

When the same variable is defined in multiple places, this is the winner:

```
1. Command-line args  (-e)
2. Task level         (vars: inside a task)
3. vars_files
4. vars_prompt
5. Play level         (vars: at play level)
6. Inventory / group_vars
```

---

## Data Types

```yaml
  vars:
    course: "DevSecOps with AIOps"   # string
    duration_in_hrs: 150             # number (int)
    is_live_sessions: true           # boolean
    topics:                          # list
    - linux
    - shell
    - ansible
    more_details:                    # map (key-value pairs)
      months: 5
      trainer: "Sivakumar Reddy"
```

Access map values: `{{ more_details.trainer }}`  
Access list values: `{{ topics[0] }}`

---

## Conditions

Use `when:` to run a task only if a condition is true. No brackets needed — it's a Jinja2 expression.

```yaml
  vars:
    live_sessions: 4
  tasks:
  - name: more than 5 sessions?
    debug:
      msg: "Yes, Ansible took more than 5 sessions"
    when: live_sessions > 5
```

Even / odd check:

```yaml
  vars:
    given_number: 7
  tasks:
  - name: is even
    debug:
      msg: "{{ given_number }} is even"
    when: given_number % 2 == 0

  - name: is odd
    debug:
      msg: "{{ given_number }} is odd"
    when: given_number % 2 != 0
```

---

## ansible_facts

Ansible automatically collects info about the target server before running tasks — OS, IP, architecture, memory, etc. This is stored in `ansible_facts`.

```yaml
  tasks:
  - name: print all facts
    debug:
      msg: "{{ ansible_facts }}"
```

Useful for writing playbooks that work on both RedHat and Debian servers:

```yaml
  tasks:
  - name: RedHat family
    debug:
      msg: "Package manager is dnf"
    when: ansible_facts.os_family == "RedHat"

  - name: Debian family
    debug:
      msg: "Package manager is apt-get"
    when: ansible_facts.os_family == "Debian"
```

`ansible_facts` is a **reserved special variable** — Ansible fills it automatically, you just use it.

---

## Loops

When you need to do the same thing for multiple items, use `loop:`. The current item is always available as `{{ item }}`.

### Simple list

```yaml
  tasks:
  - name: greet everyone
    debug:
      msg: "Hi {{ item }}"
    loop:
    - Ramesh
    - Raheem
    - John
```

### Install multiple packages

```yaml
- name: install packages
  hosts: frontend
  become: yes        # run as sudo
  tasks:
  - name: install packages
    package:
      name: "{{ item }}"
      state: present
    loop:
    - nginx
    - mysql
    - mongodb-mongosh
    - git
```

### Loop with maps — different state per item

When each item needs its own properties, use inline maps with `item.key`:

```yaml
  tasks:
  - name: manage packages
    package:
      name: "{{ item.name }}"
      state: "{{ item.state }}"
    loop:
    - { name: "nginx", state: "present" }   # install
    - { name: "mysql", state: "absent" }    # remove
    - { name: "zip",   state: "present" }   # install
```

---

## Quick Reference

| Concept | One-liner |
|---------|-----------|
| Configuration management | Managing server state through code, not manually |
| Push architecture | You push changes to servers (Ansible's model) |
| Pull architecture | Servers pull changes from a central server (Chef/Puppet) |
| Inventory | File listing which servers Ansible should manage |
| Ad-hoc command | One-off Ansible task run from the terminal |
| Playbook | YAML file with a list of tasks to run on target servers |
| Module | Pre-built unit of work in Ansible (`ping`, `dnf`, `copy`, etc.) |
| `vars` | Define variables at play or task level |
| `vars_files` | Load variables from an external YAML file |
| `vars_prompt` | Prompt user for variable input at runtime |
| `group_vars/` | Auto-loaded variables scoped to an inventory group |
| `-e` | Command-line variable — highest priority |
| `when:` | Run a task conditionally |
| `ansible_facts` | Auto-collected system info (OS, IP, arch, etc.) |
| `loop:` | Repeat a task over a list; current item is `{{ item }}` |
| `become: yes` | Run task with sudo / privilege escalation |
