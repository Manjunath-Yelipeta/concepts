1. [AWS — Regions and Availability Zones](#aws--regions-and-availability-zones)
2. [Linux](#linux)
3. [Authentication and SSH](#authentication-and-ssh)
4. [Firewall and Security Groups](#firewall-and-security-groups)
5. [Linux File System](#linux-file-system)
6. [Linux Commands](#linux-commands)
7. [Vim Editor](#vim-editor)
8. [Linux User Management](#linux-user-management)
9. [File Permissions and Ownership](#file-permissions-and-ownership)
10. [SSH Configuration and Key-Based Auth Setup](#ssh-configuration-and-key-based-auth-setup)
11. [tar — Archiving and Compression](#tar--archiving-and-compression)
12. [find — Searching Files](#find--searching-files)
13. [Removing a User Safely](#removing-a-user-safely)
14. [Package Management](#package-management)
15. [Service Management](#service-management)
16. [Process Management](#process-management)
17. [Network Management](#network-management)

## AWS — Regions and Availability Zones

Your AWS servers physically exist inside data centers around the world. AWS organizes these into two levels:

**Region** — a geographic area. Examples:
- `us-east-1` → North Virginia, USA — cheapest region; new AWS services always launch here first
- `ap-south-1` → Mumbai, India
- `eu-west-1` → Ireland

**Availability Zone (AZ)** — an isolated data center inside a region. Every region has at least 2 AZs; some have 6.

Think of it this way: **Hyderabad is a Region**. North Hyderabad is AZ-1 and South Hyderabad is AZ-2. If AZ-1 has a power failure or a flood, your application is still running from AZ-2. This is called **high availability** — your service stays up even when one data center goes down.

When you build applications on AWS, you always spread them across multiple AZs for this reason. A single-AZ deployment is a single point of failure.

> In this course we will mostly use `us-east-1` — it is the cheapest region and has all the latest features.

---

## Linux

### Why Linux, Not Windows?

All serious cloud and DevOps work runs on Linux. Here is the direct comparison:

| | Windows | Linux |
|---|---|---|
| Interface | Heavy GUI — high RAM and CPU usage | Lightweight — runs on minimal resources |
| Security | More targeted by attackers | More secure by design |
| Cost | Expensive licenses | Open source — mostly free |
| Server use | Built for desktops | Built for long-running servers |
| Remote access | RDP (heavy) | SSH (lightweight terminal) |

**How Linux came to exist:** IBM built the first PCs and servers by tightly coupling hardware and software — you had to buy both together. Linus Torvalds (who later also created Git) looked at Unix's principles and rewrote an operating system from scratch in C that could run on any hardware. That became Linux.

Linux is technically just the **kernel** — the part that sits between hardware and software, managing memory, CPU, and storage. Everything else (the shell, the file system, the commands) is built on top of it.

### Distributions (Distros)

Different organizations package the Linux kernel with their own tools and support — these are called **distributions**:

- **RedHat family**: RHEL, CentOS, Fedora, Amazon Linux — commands are nearly identical across all of them
  - Paid support available: if something breaks on an enterprise RHEL server, RedHat engineers join your team and fix it
  - Community support: open source versions like CentOS get community help instead
- **Debian family**: Ubuntu, Debian

In this course we use **Amazon Linux 2023**, which is RedHat-based. Everything you learn here works on CentOS, RHEL, and most other Linux servers.

### Users and the Command Prompt

Linux has two types of users:

- **Normal user** — prompt ends with `$` — limited permissions, cannot break the system
- **Root / Admin user** — prompt ends with `#` — full system access, can do anything

To temporarily switch to root:
```bash
sudo su
```

Use root carefully. One wrong `rm` command as root and there is no undo.

### Paths — Absolute vs Relative

Every file and directory on Linux has an address. There are two ways to refer to it:

- **Absolute path** — starts from `/` (root), works from anywhere:
  `/home/ec2-user/devops/repos`

- **Relative path** — relative to your current location:
  `repos/notes` (only works if you are already inside `/home/ec2-user/devops`)

```bash
pwd                  # print working directory — shows exactly where you are right now
cd /home/ec2-user    # go to that location (absolute — works from anywhere)
cd repos             # go into repos/ from your current location (relative)
cd ..                # go up one level
```

### Command Structure

Every Linux command follows this pattern:

```
command  [options]  [arguments]
```

Options modify how the command behaves. They come in two forms:
- Short: `-l` (single dash + one letter — quick to type)
- Long: `--list` (double dash + full word — more readable in scripts)

You can combine short options: `ls -ltr` is the same as `ls -l -t -r`.

---

## Authentication and SSH

### The Three Ways to Prove Who You Are

Before connecting to any server, you must prove your identity. Authentication falls into three categories:

1. **What you know** — username and password. Simple, but can be guessed, phished, or stolen.
2. **What you have** — a token or key file. Examples: Google Authenticator OTP, RSA token, SSH key pair.
3. **What you are** — biometrics. Fingerprint, retina scan, face recognition.

For Linux servers on AWS, we use **SSH key pairs** (category 2). Far more secure than passwords over the internet.

### Generating an SSH Key Pair

```bash
ssh-keygen -f my-key-name
```

This creates two files:
- `my-key-name` → **private key** — stays on your laptop. Never share this with anyone, ever.
- `my-key-name.pub` → **public key** — goes onto the server.

When you connect, SSH proves you hold the private key without ever transmitting it over the network. Even if someone captures all your traffic, they cannot steal your key.

### Connecting to an AWS Server

```bash
ssh -i <private-key-file> ec2-user@<server-public-ip>
```

- `ec2-user` is the default username on Amazon Linux (Ubuntu uses `ubuntu`, CentOS uses `centos`)
- `-i` points to your private key file
- Get the public IP from your AWS EC2 console

**When something goes wrong — follow this order:**
1. Read the error message carefully — it usually tells you what is wrong
2. Search Google or ChatGPT with the exact error text
3. Post in the class WhatsApp/Slack group
4. Bring it to the QA session

---

## Firewall and Security Groups

A **firewall** decides which network traffic is allowed in and out. On AWS, this is a **Security Group** — a virtual firewall attached to each EC2 instance.

- **Inbound / Ingress** — traffic coming *into* your server. You must explicitly allow each port.
- **Outbound / Egress** — traffic going *out* from your server. Allowed by default.

Every network connection is identified by three things: **protocol + IP address + port number**.

Common ports to know:

| Protocol | Port | Used For |
|---|---|---|
| SSH | 22 | Remote terminal access to Linux servers |
| HTTP | 80 | Web traffic (unencrypted) |
| HTTPS | 443 | Web traffic (encrypted) |
| MySQL | 3306 | Database connections |
| FTP | 21 | File transfers |

There are 65,536 total ports (0–65535). Applications listen on specific ports. If a port is not open in the Security Group, the connection is blocked before it reaches your application — it just times out with no explanation to the person connecting.

---

## Linux File System

Linux has a single root `/` — everything branches from here. There is no `C:\` or `D:\` like Windows.

| Directory | What Lives Here |
|---|---|
| `/` | Root — the starting point of everything |
| `/home` | Home directories for each user — e.g., `/home/ec2-user` |
| `/etc` | System and application configuration files |
| `/tmp` | Temporary files — cleared on reboot |
| `/bin` | Core commands and binaries — ls, cat, grep, etc. |

When you log in, you land in your home directory: `/home/ec2-user` on Amazon Linux. The shortcut `~` always refers to your own home directory regardless of username.

---

## Linux Commands

### Listing Files

```bash
ls                # list files and directories
ls -l             # long format — shows permissions, owner, size, modification date
ls -ltr           # sorted by time, newest at the bottom (very useful for log files)
ls -la            # include hidden files (names starting with .)
```

In `ls -l` output, the first character tells you the type:
- `d` → directory
- `-` → regular file

Hidden files start with `.` — like `.bashrc`, `.ssh/`. They exist to reduce clutter. The `-a` flag reveals them.

---

### System Information

```bash
uname -a          # print all system information — kernel version, architecture, hostname
```

---

### Creating Files and Directories

```bash
touch notes.txt             # create an empty file (safe to run on existing files — just updates the timestamp)
mkdir devops                # create a directory
mkdir -p devops/projects    # create nested directories — no error if they already exist
```

---

### Reading Files

```bash
cat notes.txt               # print entire file to the screen
cat > notes.txt             # write to a file interactively — type your content, press Ctrl+D when done
cat >> notes.txt            # append to a file without overwriting what is already there
cat file1 file2 > file3     # merge file1 and file2 into a new file3
```

The `>` operator redirects output into a file — **overwrites** existing content.
The `>>` operator appends to a file — **adds to** existing content.

---

### Copying, Moving, Deleting

```bash
cp source.txt destination.txt       # copy a file
cp -r source-dir/ dest-dir/         # copy a directory and everything inside it

mv oldname.txt newname.txt          # rename a file
mv file.txt /tmp/                   # move a file to another location

rm file.txt                         # delete a file
rm -r my-folder/                    # delete a directory and all its contents

rmdir devops                        # delete a directory — only works if it is empty
```

> `rm -r` is permanent. Linux has no Recycle Bin. Before deleting, confirm your location with `pwd` and check what you are targeting with `ls`.

---

### Downloading Files

```bash
wget https://example.com/file.zip           # download a file and save it to disk
curl https://example.com/api/health         # fetch content and print to screen
curl -s https://example.com/file.txt        # silent mode — no progress bar
```

`wget` is for downloading and saving files. `curl` is for fetching content directly — you will use it constantly for APIs, health checks, and scripting.

---

### grep — Searching Inside Files

`grep` finds lines that match a pattern. One of the most-used commands in DevOps.

```bash
grep "error" app.log              # lines containing "error"
grep -i "error" app.log           # case-insensitive — matches error, Error, ERROR
grep -n "error" app.log           # show the line number alongside each match
grep -c "error" app.log           # count how many lines match
grep -v "error" app.log           # show lines that do NOT match (exclude/invert)
```

**Remember:** Linux is case-sensitive everywhere. `Linux` and `linux` are two different things. Always use `-i` when you are not sure of the case.

---

### Piping — Chaining Commands Together

The `|` (pipe) sends the output of one command as input into the next. This is one of the most powerful ideas in Linux — combine simple tools to do complex work.

```bash
curl -s https://raw.githubusercontent.com/daws-90s/notes/main/session-02.txt | grep "linux"

ls -la | grep ".txt"
```

Think of it as: "take the output of the left side, and feed it directly into the right side." No temporary files needed.

---

### head and tail — Reading Parts of Large Files

```bash
head app.log              # first 10 lines
head -n 20 app.log        # first 20 lines

tail app.log              # last 10 lines
tail -n 20 app.log        # last 20 lines
tail -f app.log           # keep printing new lines as they are added (live follow)
```

`tail -f` is what you use when watching a running application's logs in real time — every new log line appears on your screen as it is written.

---

### cut — Extract Specific Fields

`cut` extracts columns from structured text using a delimiter.

```bash
# /etc/passwd holds all Linux user accounts
# Format per line: username:password:uid:gid:info:home-dir:shell

cut -d ":" -f1 /etc/passwd          # extract field 1 — username
cut -d ":" -f3 /etc/passwd          # extract field 3 — user ID
cut -d ":" -f1,3 /etc/passwd        # extract fields 1 and 3 together
```

`-d` sets the delimiter character. `-f` picks which field(s) to keep.

---

### awk — Smarter Text Processing

`awk` goes further than `cut`. It can filter rows by condition, work with the last field dynamically, and do calculations.

```bash
# Extract just the filename from a full URL
echo "https://raw.githubusercontent.com/daws-90s/notes/main/session-02.txt" | awk -F "/" '{print $NF}'
# Output: session-02.txt

# Print usernames from /etc/passwd
awk -F ":" '{print $1}' /etc/passwd

# Print only users with UID greater than 999 — these are regular users, not system accounts
awk -F ":" '$3 > 999 { print $1 }' /etc/passwd
```

`-F` sets the field separator. `$NF` means the last field (NF = number of fields). `$1`, `$2`, `$3` are fields by position.

---

### Quick Reference

```
Navigation       pwd, cd, ls
System info      uname -a
File ops         touch, cat, cp, mv, rm, mkdir, rmdir
Download         wget, curl
Search           grep
Text processing  cut, awk
Log reading      head, tail, tail -f
Piping           |
Redirection      >, >>
Privileges       sudo su
Editor           vim
User mgmt        useradd, groupadd, usermod, passwd, id
Permissions      chmod, chown
SSH config       sshd -t, systemctl restart sshd
```

---

## Vim Editor

`vim` is the standard terminal text editor on Linux servers. When there is no GUI, vim is how you edit files.

```bash
vim filename.txt    # open file (creates it if it does not exist)
```

Vim has two working modes:

### Esc Mode (Navigation and Actions)

Press `Esc` to enter this mode. Used for moving around and performing actions on text.

```
gg          go to the top of the file
Shift+G     go to the bottom of the file
u           undo the last change
Ctrl+R      redo
yy          copy the current line
p           paste below the current line
Shift+P     paste above the current line
10p         paste 10 times
```

### Colon Mode (Commands)

Type `:` from Esc mode to enter colon mode. Used for saving, searching, and replacing.

```
:w              save the file
:wq             save and quit
:q              quit (only works if there are no unsaved changes)
:q!             force quit — discard all changes

:set nu         show line numbers
:set nonu       hide line numbers
:20             jump to line 20

:/word          search forward for "word" — press n for the next match
:?word          search backward for "word"
:noh            clear the search highlight

:4d             delete line 4
:2,5d           delete lines 2 to 5
:%d             delete all lines in the file

:4s/old/new         replace first occurrence of "old" with "new" on line 4
:4s/old/new/g       replace all occurrences on line 4
:%s/old/new         replace first occurrence on every line
:%s/old/new/g       replace all occurrences everywhere in the file
```

> To type text, press `i` to enter Insert mode. When done, press `Esc` to go back to Esc mode, then `:wq` to save and exit.

---

## Linux User Management

On a real server, you never give everyone the root login. You create separate users, assign them to groups, and control exactly what each group is allowed to do. This is the same idea as IAM (Identity and Access Management) in AWS.

```
User  → an individual account (ramesh, john, deploy-bot)
Group → a collection of users (devops-team, developers, admins)
Role  → the level of access (read-only, read-write, admin)
```

A real example:

| Group | Access |
|---|---|
| `devops-trainee` | Read only |
| `devops-juniors` | Read + limited writes |
| `devops-seniors` | Read + write + update |
| `devops-leads` | Full — read, write, update, delete |

### Creating Users and Groups

Every Linux user must belong to exactly one **primary group** and can optionally belong to additional **secondary groups**.

```bash
useradd ramesh              # create user — also creates a group named "ramesh"
id ramesh                   # show user ID, primary group, and all secondary groups
passwd ramesh               # set or change the password for ramesh

groupadd devops-team        # create a new group
```

User information is stored in two files:
- `/etc/passwd` — all users and their home directories
- `/etc/group` — all groups and their members

### Managing Group Membership

```bash
usermod -g devops-team ramesh       # set devops-team as ramesh's primary group
usermod -G devops-team ramesh       # set devops-team as a secondary group (replaces existing secondary groups)
usermod -aG devops-team ramesh      # append devops-team as a secondary group (safe — does not remove existing ones)
```

Use `-aG` when adding a user to a new group — it appends rather than replaces.

### Giving Sudo Access

`sudo` lets a normal user run commands as root. There are two ways to grant it:

**Option 1 — Add to the wheel group** (simplest):
```bash
usermod -aG wheel ramesh     # ramesh can now run any command with sudo
```

**Option 2 — Custom group permissions in `/etc/sudoers`** (fine-grained):
```bash
# Add this line to /etc/sudoers to allow the devops-user-managers group
# to only run useradd and usermod — nothing else
%devops-user-managers    ALL=(ALL)    /usr/sbin/useradd, /usr/sbin/usermod
```

To remove a user from a group:
```bash
gpasswd -d ramesh wheel     # remove ramesh from the wheel group
```

---

## File Permissions and Ownership

Every file and directory on Linux has three permission sets and an owner.

### Reading Permission Notation

```
-rw-r--r--  1  ramesh  devops  1024  May 7 10:00  notes.txt
```

Breaking down `-rw-r--r--`:

```
-   rw-       r--        r--
|   |         |          |
|   owner(u)  group(g)   others(o)
|
file type: - = file, d = directory
```

Each position can be `r` (read), `w` (write), `x` (execute), or `-` (no permission).

### chmod — Changing Permissions

**Symbolic method** — add or remove specific permissions:
```bash
chmod g+w notes.txt      # give the group write permission
chmod o-r notes.txt      # remove read permission from others
chmod u+x deploy.sh      # give the owner execute permission
```

**Numeric method** — set all permissions at once using numbers:

| Permission | Value |
|---|---|
| r (read) | 4 |
| w (write) | 2 |
| x (execute) | 1 |

Add the values for each group — owner, group, others — and combine:

```bash
chmod 640 notes.txt     # owner: rw- (6), group: r-- (4), others: --- (0)
chmod 700 .ssh          # owner: rwx (7), group: --- (0), others: --- (0)
chmod 600 authorized_keys   # owner: rw- (6), group: --- (0), others: --- (0)
```

Common combinations: `755` (owner full, everyone else read+execute), `644` (owner read+write, everyone else read-only), `600` (owner only).

### chown — Changing Ownership

Only the root user can change file ownership.

```bash
chown ramesh notes.txt              # change owner to ramesh
chown ramesh:devops notes.txt       # change owner to ramesh and group to devops
chown -R ramesh:ramesh /home/ramesh # change owner recursively for entire directory
```

`-R` applies the change to everything inside the directory, including all subdirectories and files.

---

## SSH Configuration and Key-Based Auth Setup

### SSH Server Configuration

The SSH server's behavior is controlled by `/etc/ssh/sshd_config`. This is where you enable or disable password login, change the default port, and more.

```bash
# By default, Amazon Linux disables password authentication
# This line in /etc/ssh/sshd_config controls it:
PasswordAuthentication no
```

After editing `sshd_config`, always check the syntax before restarting:
```bash
sshd -t                     # test config file for syntax errors — no output means it is clean
systemctl restart sshd      # apply the changes by restarting the SSH service
```

If `sshd -t` reports an error, fix it before restarting — a broken config can lock everyone out of the server.

### Setting Up Key-Based Auth for a New User

When you create a new user (e.g., `ramesh`) and want them to connect via SSH key:

1. **The user generates their key pair** on their own machine:
   ```bash
   ssh-keygen -f ramesh-key
   ```

2. **The user sends only their public key** (`ramesh-key.pub`) to the admin. Never the private key.

3. **The admin sets up the `.ssh` directory** on the server under the user's home:
   ```bash
   mkdir /home/ramesh/.ssh
   touch /home/ramesh/.ssh/authorized_keys
   # paste the public key content into authorized_keys
   ```

4. **Set the correct permissions** — SSH will refuse to use keys if permissions are too open:
   ```bash
   chmod 700 /home/ramesh/.ssh
   chmod 600 /home/ramesh/.ssh/authorized_keys
   chown -R ramesh:ramesh /home/ramesh/.ssh
   ```

5. **The user connects**:
   ```bash
   ssh -i ramesh-key ramesh@<server-ip>
   ```

The permission and ownership requirements for `.ssh` are strict by design — if anyone other than the owner can read or write those files, SSH treats the keys as compromised and refuses the connection.

---

## tar — Archiving and Compression

Linux uses `.tar.gz` as its standard archive format (Windows uses `.zip`).

```
.zip    → Windows format
.tar.gz → Linux standard format
```

**Create an archive:**

```bash
tar -czf archive.tar.gz files-or-folders/
```

| Flag | Meaning |
|---|---|
| `c` | Create a new archive |
| `z` | Compress using gzip (`.gz`) |
| `f` | Next argument is the filename |
| `v` | Verbose — print each file as it is added |

**Extract an archive:**

```bash
tar -xzf archive.tar.gz               # extract into current directory (creates a subfolder)
tar -xzf archive.tar.gz -C /app       # extract into /app (creates a subfolder inside)
tar -xzf archive.tar.gz -C /app --strip-components=1   # extract files directly, no subfolder
```

**Exclude a folder while creating:**

```bash
tar -czf backend.tar.gz --exclude=node_modules .
```

> Always `cd` into the folder before archiving with `.` — this avoids the parent folder being baked into the archive, so extraction lands files directly without a subfolder.

---

## find — Searching Files

`find` searches the filesystem for files and directories matching conditions.

```bash
find <where-to-search> <conditions>
```

```bash
find / -iname "*.log"              # find all .log files anywhere (case-insensitive)
find /etc -name "nginx.conf"       # find a specific file under /etc
find / -type d -name "devops"      # find a directory named "devops"
find /app -type f -name "*.js"     # find all .js files under /app
find /tmp -mtime +7                # find files not modified in more than 7 days
```

| Flag | Meaning |
|---|---|
| `-name` | Match filename (case-sensitive) |
| `-iname` | Match filename (case-insensitive) |
| `-type f` | Regular files only |
| `-type d` | Directories only |
| `-mtime +N` | Modified more than N days ago |

---

## Removing a User Safely

Never just delete a user without following a proper process — they may be logged in or running processes.

**Step-by-step:**

1. **Check if the user is logged in:**
   ```bash
   who | grep ramesh
   ```

2. **Kill all their running processes:**
   ```bash
   pkill -u ramesh
   ```

3. **Remove them from all groups:**
   ```bash
   gpasswd -d ramesh <groupname>
   ```

4. **Check for sudoers entries:**
   ```bash
   grep ramesh /etc/sudoers
   ls /etc/sudoers.d/
   ```

5. **Back up their home directory:**
   ```bash
   tar -czf /backup/ramesh-home.tar.gz /home/ramesh
   ```

6. **Delete the user and their home directory:**
   ```bash
   userdel -r ramesh
   ```

`userdel -r` removes the user account and their home directory together. Without `-r`, the home directory is left behind.

---

## Package Management

On RedHat-based systems (RHEL, CentOS, Amazon Linux), the package manager is `dnf`. On Debian/Ubuntu it is `apt-get`. The concepts are identical — only the command name differs.

```bash
dnf install <package-name>       # install a package
dnf remove <package-name>        # uninstall a package
dnf update <package-name>        # update a specific package
dnf search <package-name>        # search available packages by name
dnf info <package-name>          # show details about a package

dnf list installed               # all installed packages
dnf list installed | grep nginx  # check if a specific package is installed
dnf list available               # packages available in repos but not yet installed
dnf repolist                     # list all configured repositories
```

**Module management** — some packages come in multiple versions via modules:

```bash
dnf module list                  # list all available modules and versions
dnf module disable nodejs -y     # disable the default version
dnf module enable nodejs:20 -y   # enable a specific version
dnf install nodejs -y            # install the enabled version
```

---

## Service Management

A **service** (also called a daemon) is a process that runs continuously in the background — `sshd`, `nginx`, `mysqld`, `backend` are all services. `systemctl` is the tool to manage them.

```bash
systemctl start nginx        # start the service now
systemctl stop nginx         # stop the service now
systemctl restart nginx      # stop then start (apply config changes)
systemctl status nginx       # show whether it is running and recent logs
systemctl enable nginx       # auto-start on boot — survives a reboot
systemctl disable nginx      # remove auto-start on boot
```

**Always enable a service after starting it** — if you only `start` without `enable`, the service will stop the next time the server reboots.

To check logs of a service:

```bash
journalctl -u nginx -f       # follow live logs
journalctl -u nginx --since "10 min ago"
```

---

## Process Management

Every running program on Linux is a **process**. Each process gets a unique **PID** (Process ID). Processes can spawn child processes — each child knows its parent via **PPID** (Parent Process ID).

```bash
ps                    # list processes for the current logged-in user
ps aux                # list all processes with username, CPU, and memory
ps -ef                # list all processes with PID and PPID
ps -ef | grep nginx   # find a specific process by name
```

**Background vs Foreground:**
- **Foreground** — the process holds the terminal; you cannot type other commands until it finishes
- **Background** — runs independently; terminal is free immediately

```bash
kill <PID>       # graceful shutdown — asks the process to stop cleanly
kill -9 <PID>    # force kill — immediately terminates with no cleanup
pkill -u ramesh  # kill all processes owned by user ramesh
```

> Use `kill` (graceful) first. Only use `kill -9` if the process does not respond — it gives the process no chance to clean up open files or connections.

---

## Network Management

```bash
netstat -lntp     # list all open ports with the process listening on each
```

| Flag | Meaning |
|---|---|
| `l` | Listening ports only |
| `n` | Show port numbers (not service names) |
| `t` | TCP connections only |
| `p` | Show the PID and process name |

**Typical troubleshooting pattern:**

```bash
systemctl status nginx          # is the service running?
netstat -lntp | grep 80         # is port 80 actually open?
ps -ef | grep nginx             # is the nginx process alive?
```

Use these three together to diagnose any service that is not responding.
