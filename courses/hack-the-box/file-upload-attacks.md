# File Upload Attacks
Hack The Box: Bug Bounty Hunter Path

## 1. Overview
- Topic(s) covered: File upload vuln, bypassing file filters, combining other vulns
- Goal: Learn how attackers exploit improperly secured file upload functionalities to execute malicious actions like remote code execution or privilege escalation.


## 2. File Upload Vulnerability
(File Upload Attacks occur when attackers exploit insecure file upload mechanisms to upload malicious files, such as scripts, malware, or payloads.)

### 2.1 Entry Points
- File upload forms (e.g., profile picture, file attachment)
- API endpoint that accepts file upload

### 2.2 Types of Attacks
- Remote Command Execution (RCE):
    Uploading a web shell or reverse shell script to execute commands on the server.
- Exploiting Other Vulnerabilities:
    Injecting malicious code to trigger vulnerabilities like XSS (Cross-Site Scripting) or XXE (XML External Entity Injection).
    - Examples:
        1. File metadata (XSS)
        ```bash
        $ exiftool -Comment=' "><img src=1 onerror=alert(window.origin)>' payload.jpg
        ```
        2. SVG (XSS)
        ```xml
        <?xml version="1.0" encoding="UTF-8"?>
        <!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
        <svg xmlns="http://www.w3.org/2000/svg" version="1.1" width="1" height="1">
            <rect x="1" y="1" width="1" height="1" fill="green" stroke="black" />
            <script type="text/javascript">alert(window.origin);</script>
        </svg>
        ```
        3. Read critical files (XXE)
        ```xml
        <?xml version="1.0" encoding="UTF-8"?>
        <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
        <svg>&xxe;</svg>
        ```
        4. Read source code (XXE)
        ```xml
        <?xml version="1.0" encoding="UTF-8"?>
        <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=index.php"> ]>
        <svg>&xxe;</svg>
        ```
- Denial of Service (DoS):
    Uploading excessively large files to overwhelm server resources.(e.g., ZIP bomb)
- File Overwrite:
    Replacing critical system files or configurations, potentially causing service disruptions or enabling further exploitation.

### 2.3 Types of filters
1. Clients-side validation
    - This type of validation is performed on the user's browser using JavaScript or HTML attributes to enforce restrictions, such as file type or size.
    - Example:
        1. Accepting files with `.jpg`, `.png`, or `.gif` extensions using the accept attribute in an HTML `<input>` tag.
        2. Using JavaScript to check the file's size or extension before submission.
    - Weakness:
        1. Client-side validation can be easily bypassed because it relies on the user's browser, which is under the attacker's control. An attacker can:
            - Modify the HTML form using web inspection tools (e.g., browser developer tools).
            - Disable JavaScript entirely or manipulate the validation logic in the browser.
2. Blacklist filters
    - This approach blocks specific file extensions or types known to be malicious, such as `.php`, `.jsp`, `.exe`, or `.bat`.
    - Example:
        1. A blacklist might reject any file with extensions like `.php` or `.exe`, assuming this will prevent script execution.
    - Weakness:
        1. Blacklists are prone to bypasses because they depend on explicitly listed bad extensions. Attackers can:
            - Fuzz extensions (e.g., `.php`,` .php3`, `.phtml`, or even `.php.jpg`) to find unblocked variations.
            - Rename malicious files to unlisted extensions.
            - Exploit edge cases where the server processes extensions differently from the filter (e.g., double extensions like `file.jpg.php`).
3. Whitelist filters
    - This approach only allows files with explicitly approved extensions or types, such as .jpg, .png, .pdf, or .txt.
    - Example:
        1. A whitelist might allow only .jpg or .png uploads for a profile picture functionality.
    - Weakness:
        1. Vulnerable to polyglot files (e.g., an image containing malicious code).
        2. Fails with weak MIME type checks or poorly implemented content validation.
        3. Can be bypassed using character injection (e.g., shell.php%00.jpg, shell.aspx:.jpg).
            - Common Characters for Injection
                1. `%20` (space)
                2. `%0a` (newline)
                3. `%00` (null byte)
                4. `%0d0a` (carriage return + newline)
                5. `/` (directory separator)
                6. `.\` (dot + backslash)
                7. `.` (single dot)
                8. `…`(unicode ellipsis)
                9. `:` (colon)
4.  Type filters
    - This filter validates files based on their MIME type (e.g., `image/png` or ``application/pdf``) or their content signature (magic bytes).
    - Example:
        1. A server may check for specific MIME types using headers sent with the file upload (e.g., `Content-Type: image/png`).
        2. The server may also inspect the file's magic bytes to verify its type matches the expected format.
    - Weakness:
        1. Modifying the Content-Type header using tools like Burp Suite to mimic an allowed type.
        2. Changing the magic bytes of a malicious file to match the signature of an allowed type (e.g., modifying a PHP file's signature to mimic a JPEG).
        3. Using polyglot files, where a file satisfies multiple type checks (e.g., a JPEG with embedded PHP code).

### 2.3 Tools Used
- [Burp Suite]: Web proxy to intercept and modify file upload requests.
- [ExifTool]: Embed malicious payloads in image metadata. (e.g., comment in a JPEG image)
- [OWASP ZAP]: Scan file upload functionalities for vulnerabilities.
- [Wordlists]: Compilation of web extensions to fuzz the file upload mechanism (e.g., [Seclists](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/web-extensions.txt))

## 3. Mitigation Strategies
- File Type Whitelisting:
    - Only allow specific, safe file types (e.g., .jpg, .png, .pdf).
- Content Scanning:
    - Use tools to scan files for malicious content before accepting uploads.
- Storage Configuration:
    - Store uploaded files in a directory outside the webroot to prevent direct access.
- Rename Files:
    - Rename uploaded files to remove original extensions and avoid executing scripts.
- Set Proper Permissions:
    - Limit permissions on the upload directory (e.g., disable script execution).

## 4. Resources
- [HTB Cheatsheet](./File_Upload_Attacks_Module_Cheat_Sheet.pdf): A cheatsheet containing essential web shells, bypassing techniques, and file extensions
- [PayloadsAllTheThings - File Upload](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Upload%20Insecure%20Files): Collection of payloads and techniques for file upload vulnerabilities.