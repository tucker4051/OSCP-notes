# Password Attacks — Custom Wordlists & Rules

## Overview

Most users do not choose truly random passwords.

Even when password policies require:

- uppercase letters
- lowercase letters
- numbers
- symbols
- minimum lengths

users still tend to create passwords using predictable patterns.

Common examples:

- company name + year
- pet name + number
- season + symbol
- first letter capitalized
- leetspeak substitutions
- simple monthly/yearly increments

Because of this, a **targeted custom wordlist** is often far more effective than relying only on generic lists like `rockyou.txt`.

---

# Why Custom Wordlists Matter

Generic wordlists are useful, but they do not reflect the target’s context.

A custom wordlist is often built from:

- company name
- brand names
- department names
- internal terminology
- employee names
- hobbies
- pets
- sports teams
- locations
- birthdays
- family names
- project names

This turns password cracking from blind guessing into **educated guessing**.

---

# Human Password Patterns

Even under password policy pressure, users commonly adapt simple words instead of creating complex random strings.

Common patterns:

| Description | Example |
|------------|---------|
| capitalized first letter | `Password` |
| append digits | `Password123` |
| append year | `Password2024` |
| append month | `Password08` |
| add symbol | `Password!` |
| leetspeak substitution | `P@ssw0rd` |
| combine multiple patterns | `Password2024!` |

This predictability is exactly what custom rules exploit.

---

# Where Candidate Words Come From

Potential seed words can come from:

## Personal context

- first name
- surname
- spouse / partner name
- children
- pets
- hobbies
- favourite sports
- favourite teams
- favourite bands
- favourite cities

## Company context

- company name
- product names
- values / slogans
- departments
- office location
- current projects
- company acronyms

## External sources

- LinkedIn
- Facebook
- company website
- blog posts
- press releases
- staff pages
- job descriptions
- conference bios

This is where light OSINT becomes useful.

---

# Basic Strategy

A practical custom wordlist workflow is:

1. gather target-specific words
2. clean and normalize them
3. generate variations
4. apply mutation rules
5. use the result with Hashcat or John

---

# Password Policy Awareness

If password policy is known, use it to constrain guesses.

Example policy:

- minimum 12 characters
- 1 uppercase
- 1 lowercase
- 1 digit
- 1 symbol

This immediately suggests patterns such as:

- capitalized base word
- appended birth year
- appended symbol
- company name + year
- pet name + special character

The stronger your understanding of the policy, the more precise your rules can become.

---

# Hashcat Rule Basics

Hashcat rules transform input words into candidate passwords.

Each line in a rule file is a mutation rule.

Useful functions:

| Function | Description |
|----------|-------------|
| `:` | do nothing |
| `l` | lowercase all letters |
| `u` | uppercase all letters |
| `c` | capitalize first letter |
| `sXY` | replace all `X` with `Y` |
| `$!` | append `!` |

Example rule file:

```text
:
c
so0
c so0
sa@
c sa@
c sa@ so0
$!
$! c
$! so0
$! sa@
$! c so0
$! c sa@
$! so0 sa@
$! c so0 sa@
```

---

# Generating Mutated Candidates with Hashcat

Suppose input list contains one base word:

```text
password
```

Apply custom rules:

```bash
hashcat --force password.list -r custom.rule --stdout | sort -u > mut_password.list
```

Possible output:

```text
password
Password
passw0rd
Passw0rd
p@ssword
P@ssword
P@ssw0rd
password!
Password!
passw0rd!
p@ssword!
Passw0rd!
P@ssword!
p@ssw0rd!
P@ssw0rd!
```

This is a simple but effective way to create a custom targeted list before cracking.

---

# Why `--stdout` Is Useful

`--stdout` tells Hashcat to **generate candidates without cracking**.

This is useful when you want to:

- inspect generated guesses
- build a custom wordlist first
- combine output with another tool
- sanity-check your rules before a long attack

---

# Built-In Rule Sets

Hashcat includes several ready-made rule sets, commonly located at:

```bash
/usr/share/hashcat/rules
```

List them:

```bash
ls -l /usr/share/hashcat/rules
```

Popular examples:

- `best64.rule`
- `rockyou-30000.rule`
- `dive.rule`
- `generated.rule`
- `leetspeak.rule`

---

# Best64 Rule

One of the most commonly used rule sets is:

```text
best64.rule
```

It applies high-value common transformations and is often a strong first choice after a plain dictionary attack.

Example:

```bash
hashcat -a 0 -m 0 hash.txt base_words.txt -r /usr/share/hashcat/rules/best64.rule
```

---

# Building Wordlists from Websites with CeWL

CeWL can spider a website and extract words that may be useful for password attacks.

This is especially effective against:

- company names
- products
- slogans
- leadership names
- internal terminology
- industry-specific vocabulary

Example:

```bash
cewl https://www.example.com -d 4 -m 6 --lowercase -w company.wordlist
```

Breakdown:

| Option | Purpose |
|--------|---------|
| `-d 4` | crawl depth |
| `-m 6` | minimum word length |
| `--lowercase` | normalize to lowercase |
| `-w` | output file |

Count results:

```bash
wc -l company.wordlist
```

---

# Practical Wordlist Building Workflow

## Step 1 — Collect seed words

Sources:

- company website
- staff names
- projects
- pets / family / hobbies
- locations

## Step 2 — Normalize list

Example cleanup:

```bash
sort -u seeds.txt > clean_seeds.txt
```

## Step 3 — Apply mutation rules

```bash
hashcat --force clean_seeds.txt -r custom.rule --stdout | sort -u > custom_candidates.txt
```

## Step 4 — Crack with resulting list

```bash
hashcat -a 0 -m <mode> hashes.txt custom_candidates.txt
```

---

# Example Targeting Logic

If you know a user:

- works at `Nexura`
- likes baseball
- has a pet named `Bella`
- was born in `1998`

Possible base words:

```text
nexura
bella
baseball
mark
maria
alex
francisco
august
```

Possible mutations:

```text
Bella1998!
Nexura2024!
Baseball1!
Maria08!
Alex1998!
```

This is exactly the type of thinking custom wordlists aim to formalize.

---

# Common Mutation Ideas

Useful transformations to model:

- capitalize first letter
- append current year
- append birth year
- append month/day
- append `!`
- substitute `a -> @`
- substitute `o -> 0`
- combine company + year
- combine pet + symbol
- combine spouse/child + digits

---

# When Custom Wordlists Work Best

Custom wordlists are most effective when:

- password policy is known
- some OSINT is available
- user naming patterns are known
- generic wordlists failed
- target is an employee or named individual
- company branding is likely used in passwords

---

# Limitations

Custom wordlists are still a guessing method.

They may fail if:

- user uses a password manager
- password is fully random
- policy is very strict
- no relevant OSINT exists
- target rotates passwords unpredictably

Even so, they are often far more efficient than blind brute force.

---

# Typical Workflow

1. identify target and context
2. gather seed words
3. consider password policy
4. build base list
5. build custom rules
6. generate mutated candidates
7. attack hashes with new list
8. refine based on failures

---

# Key Takeaways

- users remain predictable even under complexity requirements
- custom wordlists are often stronger than generic wordlists
- OSINT can significantly improve cracking success
- rules allow rapid expansion of likely password candidates
- `hashcat --stdout` is excellent for generating targeted lists
- CeWL is useful for harvesting company-specific words from websites

---

# Tags

#password-attacks #wordlists #rules #hashcat #osint #password-cracking #oscp