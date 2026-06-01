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
