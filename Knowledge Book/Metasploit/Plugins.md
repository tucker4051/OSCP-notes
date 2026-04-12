# Metasploit Plugins

## Overview

Metasploit plugins extend `msfconsole` by adding extra commands, integrations, and automation.

They can be used to:

- integrate external tools
- automate repetitive tasks
- manage projects
- enrich database workflows
- streamline session handling
- add custom commands to msfconsole

Plugins are different from modules:

- **modules** perform scanning, exploitation, post-exploitation, etc.
- **plugins** extend the **framework itself**

---

# 1. Why Plugins Matter

Without plugins, you may need to:

- switch between multiple tools
- repeatedly import/export results
- manually re-enter targets/options
- do repetitive session or project tasks by hand

Plugins help centralise workflows inside Metasploit.

Common benefits:

- better automation
- richer integrations
- easier project tracking
- faster post-exploitation workflows

---

# 2. Default Plugin Directory

Installed plugins are typically located at:

```bash
/usr/share/metasploit-framework/plugins
```

List them:

```bash
ls /usr/share/metasploit-framework/plugins
```

Example output may include:

```text
alias.rb
auto_add_route.rb
db_tracker.rb
libnotify.rb
msgrpc.rb
nessus.rb
nexpose.rb
openvas.rb
sqlmap.rb
wmap.rb
```

---

# 3. Loading a Plugin

Use `load` from inside `msfconsole`.

## Example — Load Nessus Plugin

```bash
load nessus
```

Expected output:

```text
[*] Nessus Bridge for Metasploit
[*] Type nessus_help for a command listing
[*] Successfully loaded Plugin: Nessus
```

---

## Plugin-Specific Help

Many plugins expose their own help command.

Example:

```bash
nessus_help
```

This will show commands specific to that plugin.

---

# 4. Plugin Load Failure

If the plugin does not exist or is not installed correctly:

```bash
load Plugin_That_Does_Not_Exist
```

Expected error:

```text
[-] Failed to load plugin ...
```

Typical reasons:

- wrong plugin name
- file not present in plugins directory
- wrong permissions
- syntax / compatibility issue

---

# 5. Installing Custom Plugins

Custom plugins are usually Ruby files (`.rb`).

Typical workflow:

1. download plugin source
2. copy `.rb` file into plugins directory
3. launch `msfconsole`
4. load plugin with `load <name>`

---

## Example — Download Plugin Collection

```bash
git clone https://github.com/darkoperator/Metasploit-Plugins
```

List contents:

```bash
ls Metasploit-Plugins
```

---

## Copy Plugin into MSF Directory

Example:

```bash
sudo cp ./Metasploit-Plugins/pentest.rb /usr/share/metasploit-framework/plugins/pentest.rb
```

---

## Load the Plugin

```bash
msfconsole -q
load pentest
```

Expected output shows successful load and plugin banner.

---

# 6. Verifying Plugin Commands

After loading a plugin, the main help menu is often extended.

Check:

```bash
help
```

You may now see new command groups added by that plugin.

Example groups from `pentest` plugin:

- Tradecraft Commands
- auto_exploit Commands
- Discovery Commands
- Project Commands
- Postauto Commands

---

# 7. Common Built-In / Popular Plugins

Examples often encountered:

| Plugin | Purpose |
|--------|---------|
| `nessus` | Nessus integration |
| `nexpose` | Nexpose integration |
| `openvas` | OpenVAS integration |
| `sqlmap` | sqlmap integration |
| `msgrpc` | remote API access |
| `auto_add_route` | automatically add pivot routes |
| `session_notifier` | notify on new sessions |
| `db_tracker` | assist DB tracking |
| `wmap` | web app mapping / web assessment support |

Other well-known names often associated with Meterpreter / post-exploitation include:

- `Incognito`
- `Railgun`
- `Stdapi`
- `Priv`
- `Mimikatz` (older/plugin-style usage or extensions)

---

# 8. Typical Plugin Use Cases

## Vulnerability Scanner Integration

Useful for importing / working from:

- Nessus
- Nexpose
- OpenVAS

This can help align Metasploit exploitation with imported scan results.

---

## Session Automation

Useful for:

- tagging sessions
- notifying on new sessions
- running multi-session commands
- route management for pivoting

---

## Project Management

Useful for:

- workspace support
- client/project tracking
- running grouped actions against current scope

---

# 9. Example — Nessus Plugin

## Load Plugin

```bash
load nessus
```

## Show Help

```bash
nessus_help
```

Typical command families may include:

- connect/login/logout
- policy listing
- policy deletion
- scan interaction
- user management

---

# 10. Example — Pentest Plugin

## Load Plugin

```bash
load pentest
```

## Check New Help Entries

```bash
help
```

Example commands added may include:

| Command | Purpose |
|--------|---------|
| `check_footprint` | check possible footprint of post module |
| `show_client_side` | show matched client-side exploits |
| `vuln_exploit` | run exploits based on scanner-imported data |
| `discover_db` | run discovery against DB hosts |
| `network_discover` | enumerate non-pivot networks |
| `pivot_network_discover` | enumerate pivotable networks |
| `multi_cmd` | run shell command against many sessions |
| `multi_post` | run post modules against many sessions |
| `sys_creds` | collect system creds from sessions |

These types of commands can save significant time.

---

# 11. Plugin Workflow

Typical workflow for using a plugin:

1. identify need
2. confirm plugin exists
3. load plugin
4. check plugin-specific help
5. run commands
6. validate impact on workspace / sessions / database

---

# 12. Practical Notes

- not all plugins are installed by default
- plugin compatibility may vary by Metasploit version
- custom plugins may break after framework updates
- always read plugin help before use
- plugin commands become part of your msfconsole workflow only after loading

---

# 13. Mixins (Concept Note)

Metasploit is written in **Ruby**, and many of its extensibility features rely on Ruby concepts such as **mixins**.

A mixin is a Ruby module included into a class to provide reusable functionality.

Why this matters conceptually:

- Metasploit can be extended cleanly
- many components share reusable methods
- framework customization is very flexible

For day-to-day exam use, you do **not** need deep mixin knowledge, but it helps explain why Metasploit is so modular.

---

# 14. Common Commands

## List plugin files

```bash
ls /usr/share/metasploit-framework/plugins
```

## Load plugin

```bash
load nessus
load pentest
```

## Check framework help

```bash
help
```

## Plugin-specific help

```bash
nessus_help
```

## Launch Metasploit quietly

```bash
msfconsole -q
```

---

# 15. Key Takeaways

- plugins extend Metasploit itself, not just a target action
- they are useful for integrations, automation, and workflow improvement
- load plugins with `load <name>`
- check new commands with `help`
- plugin-specific help is often essential
- custom plugins are usually installed by copying `.rb` files into the plugins directory

---

# Cheatsheet

## Plugin directory

```bash
ls /usr/share/metasploit-framework/plugins
```

## Load built-in plugin

```bash
load nessus
load openvas
load sqlmap
```

## Plugin help

```bash
nessus_help
help
```

## Install custom plugin

```bash
git clone https://github.com/darkoperator/Metasploit-Plugins
sudo cp ./Metasploit-Plugins/pentest.rb /usr/share/metasploit-framework/plugins/pentest.rb
msfconsole -q
load pentest
```

## Verify DB / plugin-driven workflows

```bash
db_status
workspace
help
```

---

# Related Notes

- Metasploit overview
- databases
- Meterpreter commands
- resource scripts
- jobs & sessions
- automation

---

# Tags

#metasploit #plugins #automation #nessus #nexpose #openvas #sqlmap #oscp