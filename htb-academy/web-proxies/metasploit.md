# Metasploit Through a Web Proxy — Study Notes

Reference notes for the **HTB Academy – Web Proxies** topic, covering how to run a **Metasploit
auxiliary scanner** and route its traffic through an intercepting proxy (**Burp**), using the
`robots_txt` module as the worked example.

> [!NOTE]
> These notes describe work on an authorized HTB lab. Only test applications you own or are
> explicitly permitted to test.

---

## Contents

| Part | Topics |
| ---- | ------ |
| **1. Tools** | [What is Metasploit?](#1-what-is-metasploit) · [Auxiliary modules](#2-auxiliary-modules) |
| **2. Workflow** | [Start the console](#3-start-the-console) · [Select the module](#4-select-the-module) · [Set the options](#5-set-the-options) · [Run the scan](#6-run-the-scan) |
| **3. Proxying** | [Why route through a proxy?](#7-why-route-through-a-proxy) · [The PROXIES option](#8-the-proxies-option) |
| **4. Revision** | [Command reference](#9-command-reference) · [Quick revision](#10-quick-revision) |

---

# Part 1 — Tools

## 1. What is Metasploit?

**Metasploit Framework** is a toolkit for security testing. It bundles reusable **modules** —
scanners, exploits, payloads, and helpers — behind one console, so a repeatable test becomes
"pick a module, set a few options, run it."

**`msfconsole`** is the interactive command line for the framework. Everything below happens
inside it.

---

## 2. Auxiliary Modules

**Auxiliary modules** do work that isn't itself an exploit: scanning, enumeration, fuzzing, and
information gathering. The one used here reads a site's `robots.txt`:

| Module | Purpose |
| ------ | ------- |
| `auxiliary/scanner/http/robots_txt` | Request `/robots.txt` and report the paths it lists |

A site's **`robots.txt`** tells crawlers which paths to avoid. Those "disallowed" paths are often
exactly the directories worth noting during enumeration, which is why reading the file is a common
first step.

---

# Part 2 — Workflow

## 3. Start the Console

Launch the framework console:

```text
msfconsole
```

---

## 4. Select the Module

Load the scanner with `use` and the module's full path:

```text
use auxiliary/scanner/http/robots_txt
```

Once a module is selected, the prompt changes to show it, and `show options` lists the settings it
accepts.

---

## 5. Set the Options

Each module reads its configuration from named options set with `set <NAME> <VALUE>`:

```text
set RHOST SERVER_IP
set RPORT PORT
set PROXIES HTTP:127.0.0.1:8080
```

**Breakdown:**

| Option | Meaning |
| ------ | ------- |
| `RHOST` | The target host (remote host) — here `SERVER_IP` |
| `RPORT` | The target port — here `PORT` |
| `PROXIES` | Route the module's traffic through a proxy: `HTTP:127.0.0.1:8080` |

> [!TIP]
> Replace `SERVER_IP` and `PORT` with the real values for your lab target. Capitalised
> placeholders are a reminder to substitute, not literal text.

---

## 6. Run the Scan

Execute the module:

```text
run
```

The scanner requests `/robots.txt` from the target and prints the paths it finds. `run` and
`exploit` are interchangeable for most modules.

---

# Part 3 — Proxying

## 7. Why Route Through a Proxy?

Pointing `PROXIES` at Burp (`127.0.0.1:8080`) sends every request the module makes through the
proxy first:

```text
Metasploit  ⇄  Burp (127.0.0.1:8080)  ⇄  Server
```

That means the tool's traffic shows up in Burp's **HTTP history**, so you can:

- **See exactly** what the module sent and what came back.
- **Send any request to Repeater** to probe it further by hand.
- **Confirm** the scan behaved as expected, rather than trusting the summary alone.

It connects the automated tool to the manual workflow from the Burp Repeater notes.

---

## 8. The PROXIES Option

The value is `TYPE:HOST:PORT`:

| Part | Value | Meaning |
| ---- | ----- | ------- |
| Type | `HTTP` | Proxy protocol |
| Host | `127.0.0.1` | Localhost — Burp runs on the same machine |
| Port | `8080` | Burp's default listener port |

> [!IMPORTANT]
> Burp must be listening on that address and port. If the proxy is down or on a different port,
> the module's connections fail.

---

# Part 4 — Revision

## 9. Command Reference

`msfconsole` commands used here:

| Command | Purpose |
| ------- | ------- |
| `msfconsole` | Start the Metasploit console |
| `use <module>` | Select a module to work with |
| `show options` | List the selected module's settings |
| `set <NAME> <VALUE>` | Set an option |
| `unset <NAME>` | Clear an option |
| `run` / `exploit` | Execute the module |
| `back` | Leave the current module |

---

## 10. Quick Revision

- **Metasploit** runs reusable **modules**; `msfconsole` is its command line.
- **Auxiliary** modules scan and enumerate — e.g. `scanner/http/robots_txt` reads `/robots.txt`.
- Workflow: **`use` the module → `set` RHOST / RPORT → `run`.**
- `robots.txt` lists disallowed paths, which are useful enumeration leads.
- **`PROXIES HTTP:127.0.0.1:8080`** routes the module's traffic through Burp so it appears in
  HTTP history and can be sent to Repeater.
- The `PROXIES` value is `TYPE:HOST:PORT`; the proxy must be listening or the scan fails.

---

<p align="center"><i>Practice only on systems you're authorized to test (HTB labs, localhost).</i></p>
