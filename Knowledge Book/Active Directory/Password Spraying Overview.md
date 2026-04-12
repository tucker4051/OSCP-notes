# Password Spraying Overview

## Overview

**Password spraying** is a credential access technique that attempts to authenticate to an exposed service using:

- **one common password**
- **many usernames or email addresses**

The goal is to gain access to:

- user accounts
- internal systems
- applications
- remote access portals
- an initial foothold in the target environment

Unlike traditional brute forcing, password spraying is slower and more deliberate. Instead of trying many passwords against one account, it tries **one password across many accounts**, then waits, then repeats with another password.

This is important because it reduces the chance of triggering account lockouts.

---

## Why It Matters

Password spraying is often one of the most effective ways to gain access during an internal or external assessment.

It becomes especially powerful when combined with:

- OSINT
- username enumeration
- password policy enumeration
- captured hashes
- poisoning attacks
- BloodHound analysis
- broader AD enumeration

A penetration test is never a straight line. While one technique is running in the background, such as:

- scanning
- hash cracking
- LLMNR/NBT-NS poisoning
- service enumeration

we can often use the information already collected to attempt password spraying in parallel.

Because assessments are time-boxed, this kind of overlap is often what makes the difference between a shallow and deep compromise.

---

# Story Time

## Scenario 1

In the first scenario, no easy anonymous enumeration paths were available.

There was:

- no useful SMB NULL session
- no useful LDAP anonymous bind

So the next step was to build a target username list using **Kerbrute**.

### Method used

A combined target list was created from:

- the `jsmith.txt` list from the **statistically-likely-usernames** GitHub repository
- usernames inferred from **LinkedIn** scraping

This combined list was then used with Kerbrute to:

1. enumerate valid usernames
2. spray a common password: `Welcome1`

### Result

The spray returned **two valid accounts** belonging to low-privileged users.

Although they were not highly privileged, that access was enough to:

- authenticate into the domain
- run BloodHound
- identify attack paths
- continue progressing until the domain was compromised

### Lesson

Even a very low-privileged credential can be enough to unlock the rest of the attack chain.

---

## Scenario 2

In the second scenario, standard username generation failed.

The tester tried:

- common username lists
- LinkedIn-derived usernames

but none of them produced useful results.

So the tester pivoted to **Google dorking** and searched for PDFs published by the organization.

### Key discovery

Several PDFs had metadata in the **Author** field showing that the internal username scheme used a format like:

```text
F9L8
```

This format consisted of **4-character random-looking IDs** using:

- uppercase letters `A-Z`
- digits `0-9`

Because that format was consistent, it became possible to generate **every possible combination**.

### Bash script used

```bash
#!/bin/bash

for x in {{A..Z},{0..9}}{{A..Z},{0..9}}{{A..Z},{0..9}}{{A..Z},{0..9}}
    do echo $x;
done
```

### Scale

This generated:

```text
1,679,616
```

possible usernames.

That list was then fed into Kerbrute to enumerate valid accounts across the domain.

### Result

What was intended to make usernames harder to guess actually made them **fully enumerable** because:

- the scheme was predictable
- document metadata leaked the format
- the total search space was small enough to brute through

This gave the tester a much more complete username list than usual.

Typically, a list like `jsmith.txt` might only identify **40–60%** of valid users.

In this case, the tester was able to enumerate **all or nearly all domain accounts**, dramatically increasing the odds of a successful password spray.

Eventually, valid passwords were found and the tester later chained this access into:

- **Resource-Based Constrained Delegation (RBCD)**
- **Shadow Credentials**

which ultimately led to full domain compromise.

### Lesson

Metadata leaks can massively improve spraying effectiveness.  
A predictable username scheme is often more dangerous than an obvious one.

---

# What Password Spraying Is

## Core idea

Password spraying means:

- pick one likely password
- try it against many accounts
- pause
- pick another likely password
- repeat carefully

This is different from brute force.

---

## Password Spraying vs Brute Force

### Brute force

- many passwords
- one account
- high lockout risk
- noisy
- easy to detect

### Password spraying

- one password
- many accounts
- lower lockout risk
- slower
- often more effective in enterprise environments

Think of brute force like trying every key on one door until the alarm goes off.

Password spraying is like trying the same common office key on every door in the building, one at a time, with pauses in between.

---

# Password Spray Visualization

| Attack Round | Username | Password |
|---|---|---|
| 1 | bob.smith@inlanefreight.local | Welcome1 |
| 1 | john.doe@inlanefreight.local | Welcome1 |
| 1 | jane.doe@inlanefreight.local | Welcome1 |
| DELAY |  |  |
| 2 | bob.smith@inlanefreight.local | Passw0rd |
| 2 | john.doe@inlanefreight.local | Passw0rd |
| 2 | jane.doe@inlanefreight.local | Passw0rd |
| DELAY |  |  |
| 3 | bob.smith@inlanefreight.local | Winter2022 |
| 3 | john.doe@inlanefreight.local | Winter2022 |
| 3 | jane.doe@inlanefreight.local | Winter2022 |

---

# Why Delay Matters

The delay is the entire reason password spraying is safer than brute forcing.

Without a delay, repeated failed logins may:

- increment bad password counters
- trigger lockouts
- alert defenders
- impact production users

The idea is to stay below the **account lockout threshold**.

---

# Password Spraying Considerations

## 1. Account lockouts are the biggest operational risk

A careless password spray can:

- lock out dozens or hundreds of users
- disrupt production
- create a helpdesk spike
- immediately expose the assessment

This is one of the easiest ways for a tester to cause avoidable damage.

---

## 2. Know the password policy if possible

Before spraying, try to obtain the domain password policy.

This gives you:

- minimum password length
- complexity requirements
- lockout threshold
- lockout duration
- observation/reset window

These values directly determine:

- how many passwords you can safely try
- how long to wait between rounds
- which passwords are worth attempting

---

## 3. If you do not know the policy, be conservative

If the password policy is unknown:

- avoid multiple rapid attempts
- prefer a single targeted spray
- wait hours between attempts if repeating
- consider asking the client for the policy if appropriate

A cautious rule of thumb is to wait **a few hours** between sprays if the policy is unknown.

---

## 4. Internal spraying is safer than external only when policy is known

Internal spraying may allow you to enumerate the lockout policy first, which reduces risk.

However, internal spraying can still be dangerous if:

- you guess wrong about thresholds
- multiple DCs behave differently
- fine-grained password policies exist
- lockout reset windows are misunderstood

---

## 5. One weak password can be enough

A single successful spray might only yield:

- a basic user account
- a contractor account
- a service desk user
- a shared mailbox account

That may still be enough to:

- run BloodHound
- enumerate shares
- enumerate groups
- find new attack paths
- pivot into privilege escalation later

Do not dismiss low-privileged hits.

---

# Common Real-World Password Policy Patterns

A commonly seen policy is:

- **5 bad attempts before lockout**
- **30-minute auto-unlock**
- accounts reset after a defined observation window

But not all environments follow this.

Some environments may instead have:

- stricter thresholds such as **3 attempts**
- much longer lockout periods
- **manual unlock only**
- different policies for different users/groups

This is why spraying without knowing the policy is risky.

---

# When Password Spraying Makes Sense

Password spraying is often a strong choice when:

- you have a good target user list
- you know or can estimate the password policy
- other foothold options have failed
- weak password patterns are likely
- you want a measured, high-return credential attack

It is often performed while other activities continue in parallel, such as:

- hash cracking
- poisoning attacks
- host enumeration
- service scanning

---

# Building the Username List

A successful spray starts with a strong target list.

Potential sources include:

- OSINT
- LinkedIn
- email harvesting
- SMB NULL sessions
- LDAP anonymous bind
- Kerbrute user enumeration
- BloodHound-derived identities
- PDF metadata
- naming convention analysis
- previously captured usernames from Responder/Inveigh

The better the user list, the higher the chance of success.

---

# Choosing Passwords to Spray

Good spray candidates are usually:

- common
- season-based
- company-themed
- policy-compliant
- short enough to be realistic
- complex enough to satisfy policy

Examples often seen in real environments:

- `Welcome1`
- `Password1`
- `Passw0rd`
- `Winter2022`
- `Spring2024`
- `CompanyName1`

The right choice depends on:

- password length requirements
- complexity rules
- time of year
- organization naming patterns
- known user behavior

---

# Safe Operating Guidance

## Best practices during a spray

- know the password policy if possible
- spray one password at a time
- log every attempt
- track the exact time of each round
- track which DC or service was targeted
- wait the full reset interval between rounds
- stop immediately if lockouts occur
- avoid testing accounts that appear sensitive unless explicitly in scope

## If the policy is unknown

- try one single weak/common password only
- treat it like a hail mary, not a sustained attack
- wait a long time before any second round
- avoid “just trying one more” impulsively

---

# Key Takeaways from the Two Scenarios

## Scenario 1 lesson

A common password such as `Welcome1` against a valid, curated user list can be enough to gain the first foothold.

## Scenario 2 lesson

Predictable username schemes and metadata leakage can completely transform the quality of your target list, making spraying dramatically more effective.

## Overall lesson

Password spraying is not just about passwords. It is about:

- user enumeration
- policy awareness
- timing
- discipline
- patience
- choosing the right list and the right moment

---

# Quick Reference

## Password spraying is:

- one password
- many users
- pause
- repeat carefully

## Main goal:

- gain initial access without locking accounts

## Main danger:

- causing mass account lockouts

## Main requirement:

- a strong user list and good timing

## Best supporting data:

- password policy
- username structure
- naming conventions
- OSINT
- metadata leaks

---

# Next Step

Before launching a spray, the ideal next step is to:

1. enumerate the password policy
2. build a target user list
3. choose one likely password
4. log and time your attempt carefully

After that, you can move into the practical process of:

- creating the user list
- validating usernames
- launching careful internal password sprays from Linux or Windows

---

# Tags

#active-directory  
#password-spraying  
#credential-access  
#initial-access  
#kerbrute  
#osint  
#user-enumeration  
#ad-attacks  
#internal-pentest  
#obsidian