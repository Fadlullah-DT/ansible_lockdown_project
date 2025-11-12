# Ansible Lockdown Customization

This document goes over how to customize the Ansible Lockdown **Windows-2022-CIS** and **Windows-11-CIS** roles.

---

## Table of Contents

- [Ansible Lockdown Customization](#ansible-lockdown-customization)
  - [Table of Contents](#table-of-contents)
  - [1. Overview \& Context](#1-overview--context)
  - [3. Customization via Variables (true/false toggles)](#3-customization-via-variables-truefalse-toggles)
    - [3.1 Global Flags and Sections](#31-global-flags-and-sections)
    - [3.2 Individual Rule Toggles](#32-individual-rule-toggles)
    - [3.3 Path Customization](#33-path-customization)

---

> **Important Notes (from [Lockdown Customization Docs](https://www.lockdownenterprise.com/docs/customizing-lockdown-enterprise#GH_AL_WINDOWS_2022_cis)):**
>
> - *DO NOT EDIT THE ROLE DIRECTLY*
> - *DO NOT RUN IN PRODUCTION WITHOUT TESTING IN A TEST ENVIRONMENT*
> - *DO NOT USE EXPOSED PASSWORDS*

---

## 1. Overview & Context

- The community role **ansible-lockdown** supports CIS for Windows 2022 (and Windows 11) and offers a set of variables to enable/disable controls.
- The approach: instead of editing each task, you set variables in `defaults/main.yml` (or via extra-vars) to control behavior (e.g., `rule_18_9_102_2_2=false`).

---

## 3. Customization via Variables (true/false toggles)

You can customize controls using boolean toggles instead of editing each task directly.

---

### 3.1 Global Flags and Sections

In `defaults/main.yml`, you’ll find variables like:

```yaml
section01_patch: true
section02_patch: true
...
```

You can enable or disable entire sections by setting them to `true` or `false`.
Example:

```yaml
section01_patch: true
section02_patch: true
section09_patch: true
section17_patch: true
section18_patch: false
```

---

### 3.2 Individual Rule Toggles

Within each section, you’ll find variables for specific rules such as `rule_1_2_1`, `rule_2_3_1_1`, etc.
You can disable specific controls by setting them to `false`.

```yaml
rule_1_2_1: false
rule_2_3_1_1: false
rule_9_1_1: false
rule_9_2_1: false
rule_18_9_102_2_2: false
```

> **Example:**
> `rule_18_9_102_2_2: false` setting this rule to false ensures Ansible connectivity continues without interruption.

---

### 3.3 Path Customization

Some variables in `defaults/main.yml` can be adjusted to fit your environment.
For example, **domain member maximum password age**:

```yaml
# The recommended state for this setting is: 30 or fewer days, but not 0.
win11cis_domain_member_maximum_password_age: 30
```

By default, domain members automatically change their domain passwords every 30 days

---
