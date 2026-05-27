# Shell Scripting

## What is a Shell Script?

A shell script is a file containing a series of commands that the shell reads and executes. It automates tasks that would otherwise be done manually on the command line.

```bash
#!/bin/bash       # shebang — tells the OS which interpreter to use
echo "Hello World"
```

The shebang `#!/bin/bash` must be the **first line** of every script.

---

## Problem Solving Approach

> 90% thinking, 10% coding.

Before writing any script:
1. Don't think in syntax — think in plain English first
2. Write pseudo-code (algorithm) to describe the steps
3. Translate to shell only after the logic is clear

**Example — check even or odd:**

```
number = 11
remainder = 11 % 2     # remainder = 1

if remainder == 0:
    print "even"
else:
    print "odd"
```

```bash
NUMBER=11
REMAINDER=$(( NUMBER % 2 ))

if [ $REMAINDER -eq 0 ]; then
    echo "$NUMBER is even"
else
    echo "$NUMBER is odd"
fi
```

---

## Variables

### Assigning Variables

```bash
VAR_NAME=VALUE    # no spaces around =
PERSON1=Siva
CITY="Hyderabad"
```

### Reading from User Input

```bash
echo "Enter your username:"
read USER_NAME

echo "Enter your password:"
read -s PASSWORD   # -s hides input (silent)
```

### Reading from Command Output

```bash
TIMESTAMP=$(date)
START_TIME=$(date +%s)
```

### Arithmetic

```bash
END_TIME=$(date +%s)
TOTAL=$(( END_TIME - START_TIME ))
SUM=$(( NUM1 + NUM2 ))
REMAINDER=$(( NUMBER % 2 ))
```

### DRY — Don't Repeat Yourself

Store repeated values in a variable. Change in one place, reflects everywhere.

```bash
DOMAIN="daws90s.shop"
# use $DOMAIN everywhere instead of hardcoding the string
```

---

## Command Line Arguments

Values passed to a script when running it.

```bash
sh script.sh Trump Iran
# $1 = Trump
# $2 = Iran
```

| Variable | Meaning |
|----------|---------|
| `$1`, `$2`, ... | Positional arguments |
| `$@` | All arguments |
| `$#` | Number of arguments |
| `$0` | Script name |

### shift

`shift` removes `$1` and renumbers remaining arguments — `$2` becomes `$1`, `$3` becomes `$2`, etc. Used when the first argument is a control flag and the rest are data.

```bash
sh roboshop.sh create mongodb redis mysql

ACTION=$1   # ACTION=create
shift       # $1=mongodb $2=redis $3=mysql

for instance in $@
do
    echo "Processing $instance"
done
```

---

## Special Variables

Shell automatically creates and maintains these:

```bash
echo "All args     : $@"
echo "Arg count    : $#"
echo "Script name  : $0"
echo "User         : $USER"
echo "Directory    : $PWD"
echo "Home         : $HOME"
echo "Script PID   : $$"

sleep 5 &
echo "Background PID: $!"
wait $!          # pause until that background process finishes

echo "Line number  : $LINENO"
echo "Elapsed time : $SECONDS seconds"
echo "Random number: $RANDOM"
```

---

## Exit Codes

Every command returns an exit code. Captured in `$?`.

| Code | Meaning |
|------|---------|
| `0` | Success |
| `1–127` | Failure |

```bash
ls /tmp
echo $?    # 0

ls /fakedir
echo $?    # non-zero
```

> Always capture `$?` immediately after the command — the next command will overwrite it.

---

## Data Types

Shell treats everything as a **string** by default.

```bash
NUM=10
STR="hello"
```

### Arrays

```bash
MOVIES=("RRR" "Varanasi" "Pushpa")   # index starts at 0

echo "${MOVIES[@]}"     # all elements
echo "${MOVIES[0]}"     # RRR
echo "${MOVIES[1]}"     # Varanasi
```

---

## Conditions

### Comparison Operators

| Operator | Meaning |
|----------|---------|
| `-gt` | greater than |
| `-lt` | less than |
| `-eq` | equal |
| `-ne` | not equal |
| `-ge` | greater than or equal |
| `-le` | less than or equal |

### if / elif / else

```bash
if [ $NUMBER -gt 20 ]; then
    echo "$NUMBER is greater than 20"
elif [ $NUMBER -eq 20 ]; then
    echo "$NUMBER is equal to 20"
else
    echo "$NUMBER is less than 20"
fi
```

### String Comparison

```bash
if [ "$ACTION" == "create" ]; then
    echo "creating..."
fi

if [ "$ACTION" != "delete" ]; then
    echo "not deleting"
fi
```

### Checking Root Access

```bash
USERID=$(id -u)

if [ $USERID -ne 0 ]; then
    echo "Please run this script with root access"
    exit 1
fi
```

---

## Functions

Keep repeated code in a function. Call it with arguments when needed.

```bash
FUNC_NAME() {
    # $1, $2 are arguments passed to the function
    echo "Argument 1: $1"
}

FUNC_NAME arg-1 arg-2
```

### VALIDATE Pattern

Standard pattern used across roboshop scripts to check exit codes:

```bash
VALIDATE() {
    if [ $2 -ne 0 ]; then
        echo "Installing $1 ... FAILED"
        exit 1
    else
        echo "Installing $1 ... SUCCESS"
    fi
}

dnf install nginx -y
VALIDATE nginx $?
```

---

## Loops

### For Loop — Range

```bash
for number in {1..100}
do
    echo $number
done
```

### For Loop — Arguments

```bash
for package in $@
do
    echo "Installing $package"
    dnf install $package -y
done
```

Run as: `sh script.sh mysql nginx git`

### For Loop — List

```bash
for instance in mongodb redis mysql rabbitmq
do
    echo "Processing $instance"
done
```

---

## Redirections

| Symbol | Meaning |
|--------|---------|
| `>` | Redirect output (overwrite) |
| `>>` | Redirect output (append) |
| `<` | Redirect input |
| `1>` | Redirect stdout only |
| `2>` | Redirect stderr only |
| `&>>` | Redirect both stdout and stderr (append) |

```bash
ls /tmp 1> output.log          # stdout to file
ls /fakedir 2> errors.log      # stderr to file
dnf install nginx &>> app.log  # both to file
```

---

## Logging

### tee — print to screen AND write to file

```bash
LOGS_FILE="/var/log/shell-script/app.log"

echo "Installing nginx..." | tee -a $LOGS_FILE
# -a = append (don't overwrite)
```

### Timestamp in Logs

```bash
TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")

echo "$TIMESTAMP [INFO] Starting installation" | tee -a $LOGS_FILE
echo "$TIMESTAMP [ERROR] Failed to install" | tee -a $LOGS_FILE
```

### Suppress Command Output (send to log only)

```bash
dnf install nginx -y &>> $LOGS_FILE
```

---

## Colors (ANSI Escape Codes)

```bash
R="\e[31m"   # Red
G="\e[32m"   # Green
Y="\e[33m"   # Yellow
N="\e[0m"    # Reset (no color)

echo -e "$G SUCCESS $N"
echo -e "$R FAILED $N"
echo -e "$Y SKIPPING $N"
```

> Use `echo -e` to enable escape sequence interpretation.

---

## Error Handling

### set -e

Stops the script immediately when any command fails.

```bash
#!/bin/bash
set -e

echo "Hello"
some-invalid-command    # script stops here
echo "This never runs"
```

### trap ERR

Runs a command whenever an error occurs — useful for printing which line failed.

```bash
set -e
trap 'echo "Error at line $LINENO, command: $BASH_COMMAND"' ERR
```

### Expected vs Unexpected Errors

| Type | Example | How to Handle |
|------|---------|---------------|
| Expected | Package already installed | Check before running, skip with message |
| Unexpected | Network timeout, disk full | `set -e` + `trap ERR` to catch and log |

---

## sed — Stream Editor

Edit files from the command line without opening them.

| Flag | Action |
|------|--------|
| `s/old/new/` | Substitute first match per line |
| `s/old/new/g` | Substitute all matches |
| `-i` | Edit file in-place |
| `i\text` | Insert line **before** match |
| `a\text` | Insert line **after** match |
| `d` | Delete matching line |
| `c\text` | Change (replace) matching line |
| `/pattern/d` | Delete lines matching pattern |

```bash
# Change 127.0.0.1 to 0.0.0.0 in mongod.conf
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mongod.conf

# Delete all blank lines
sed -i '/^$/d' file.txt

# Insert a comment before a matching line
sed -i '/bindIp/i\# allow remote connections' /etc/mongod.conf
```

---

## Idempotency

Running a script multiple times should produce the **same result** — no duplicate installs, no duplicate data loads.

```bash
# Check before installing
dnf list installed nginx &>> $LOGS_FILE

if [ $? -eq 0 ]; then
    echo "nginx already installed ... SKIPPING"
else
    dnf install nginx -y &>> $LOGS_FILE
fi
```

```bash
# Check before loading DB data
INDEX=$(mongosh --host mongodb.daws90s.shop --eval \
    'db.getMongo().getDBNames().indexOf("catalogue")')

if [ $INDEX -lt 0 ]; then
    mongosh --host mongodb.daws90s.shop < /app/db/master-data.js
else
    echo "Data already loaded ... SKIPPING"
fi
```

---

## Full Script Pattern

Standard structure used across all roboshop component scripts:

```bash
#!/bin/bash

# --- Setup ---
LOGS_FOLDER="/var/log/roboshop"
mkdir -p $LOGS_FOLDER
LOGS_FILE="$LOGS_FOLDER/$0.log"
TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")
R="\e[31m" G="\e[32m" Y="\e[33m" N="\e[0m"

# --- Root check ---
USERID=$(id -u)
if [ $USERID -ne 0 ]; then
    echo -e "$R Please run with root access $N"
    exit 1
fi

# --- VALIDATE function ---
VALIDATE(){
    if [ $1 -ne 0 ]; then
        echo -e "$TIMESTAMP [ERROR] $2 ... $R FAILED $N" | tee -a $LOGS_FILE
        exit 1
    else
        echo -e "$TIMESTAMP [INFO] $2 ... $G SUCCESS $N" | tee -a $LOGS_FILE
    fi
}

# --- Install with idempotency check ---
dnf list installed nginx &>> $LOGS_FILE
if [ $? -ne 0 ]; then
    dnf install nginx -y &>> $LOGS_FILE
    VALIDATE $? "Installing nginx"
else
    echo -e "$TIMESTAMP [INFO] nginx already installed ... $Y SKIPPING $N"
fi
```

---

## Scripts Reference

| Script | Concept |
|--------|---------|
| `01-hello-world.sh` | Shebang, echo |
| `02-conversation.sh` | Variables hardcoded |
| `03-variables.sh` | Variables from `$1`, `$2` |
| `04-variables.sh` | Variables from `read` |
| `05-variables.sh` | Variables from command output, epoch time |
| `06-special-vars.sh` | All special variables |
| `07-data-types.sh` | Arrays, arithmetic |
| `08-condition.sh` | if / elif / else |
| `09-install-functions.sh` | Functions, root check, VALIDATE |
| `10-logs.sh` | Logging with tee, `&>>` |
| `11-loops.sh` | For loop with range |
| `12-loops.sh` | For loop with args, timestamps |
| `13-colors.sh` | ANSI color codes |
| `14-set.sh` | `set -e` |
| `15-set.sh` | `set -e` + `trap ERR` |
