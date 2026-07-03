---
title: "Deploying React + Spring from Your Home Laptop"
date: 2024-08-18
draft: false
tags: ["react", "spring", "nginx", "windows", "https"]
categories: ["DevOps"]
description: "A complete guide to self-hosting a React + Spring web app from your home laptop using nginx on Windows, a free Let's Encrypt SSL certificate via win-acme, and a custom domain — all for the cost of electricity."
showToc: true
---

## Before You Start

1. If you want to self-host a side project from home, use a low-power PC.
2. Running a high-power gaming PC with a discrete GPU as a server is burning electricity for nothing — the idle draw alone is about the price of a chicken per month.
3. Aim for a CPU with a TDP of 15W or less. A budget laptop works great.
4. My setup: Firebat S1 Mini PC / CPU: Intel N100 (TDP 9W)
5. At an average system draw of 15W running 24/7 for a month, you consume 10.8 kWh — roughly ₩2,000.
6. Make sure VS Code, JDK, and Node.js are already installed.

## nginx

### Installation

1. Download from the [nginx download page](https://nginx.org/en/download.html).
2. Install the **Stable** version.

   ![nginx download page](/images/deploying-react-spring-from-home-laptop/2024-11-24-20-57-20.png)

3. Extract the zip to your initial CMD prompt directory, typically `C:\Users\<username>`.

   ![Extracting nginx](/images/deploying-react-spring-from-home-laptop/2024-11-24-20-58-01.png)

4. Before starting nginx, check if anything is already listening on TCP port 80:

   ```
   C:\Users\user>netstat -ano | findstr :80
     TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       12976

   C:\Users\user>taskkill /pid 12976 /f /t
   SUCCESS: The process with PID 13800 (child process of PID 12976) has been terminated.
   SUCCESS: The process with PID 12976 (child process of PID 11572) has been terminated.
   ```

5. Run the config test, then start nginx:

   ```
   C:\Users\user>cd nginx-1.26.2

   C:\Users\user\nginx-1.26.2>nginx -t
   nginx: the configuration file C:\Users\user\nginx-1.26.2/conf/nginx.conf syntax is ok
   nginx: configuration file C:\Users\user\nginx-1.26.2/conf/nginx.conf test is successful

   C:\Users\user\nginx-1.26.2>nginx
   ```

6. Visit `http://localhost` — if you see the nginx welcome page, it's running.

   ![nginx welcome page](/images/deploying-react-spring-from-home-laptop/2024-11-24-20-58-34.png)

### Configuration

1. Open `nginx.conf`:

   ![nginx.conf location](/images/deploying-react-spring-from-home-laptop/2024-11-24-20-58-58.png)

   Erase the existing content and replace it with:

   **nginx.conf**

   ```nginx
   worker_processes 1;

   error_log logs/error.log;
   pid logs/nginx.pid;

   events {
     worker_connections 1024;
   }

   http {
     server {
       listen 80;
       charset utf-8;

       location / {
         root html;
         index index.html;
       }

       location /frontend {
         proxy_pass http://localhost:10001/;
       }

       location /backend {
         proxy_pass http://localhost:20001/;
       }
     }
   }
   ```

2. Reload nginx:

   ```
   C:\Users\user\nginx-1.26.2>nginx -s reload
   ```

## React

### Creating the Project

1. Create a React project under `D:\react-spring`:

   ```
   D:\react-spring>npx create-react-app frontend
   Need to install the following packages:
   create-react-app@5.0.1
   Ok to proceed? (y) y
   ```

2. Start the dev server with `npm start`:

   ```
   D:\react-spring\frontend>npm start

   Compiled successfully!

   You can now view frontend in the browser.

     Local:            http://localhost:3000
     On Your Network:  http://192.168.0.11:3000
   ```

   ![React boilerplate](/images/deploying-react-spring-from-home-laptop/2024-11-24-20-59-28.png)

### Sample Code

1. Press `Ctrl+C` to stop the boilerplate, then trim the project down to the minimum files as shown. (`build/` and `.env` don't exist yet — that's expected.)

   ![Minimal file structure](/images/deploying-react-spring-from-home-laptop/2024-11-24-20-59-46.png)

   **public/index.html**

   ```html
   <!DOCTYPE html>
   <html lang="en">
     <head>
       <meta
         charset="UTF-8"
         name="viewport"
         content="width=device-width, initial-scale=1.0"
       />
       <title>Hello React</title>
     </head>
     <body>
       <div id="root"></div>
     </body>
   </html>
   ```

   **src/App.css**

   ```css
   .app {
     text-align: center;
     padding: 20px;
   }
   .title {
     margin-bottom: 20px;
   }
   form {
     display: flex;
     flex-direction: column;
     align-items: center;
   }
   .input-group {
     display: flex;
     flex-direction: column;
     margin-bottom: 10px;
   }
   .input-group label {
     margin-bottom: 5px;
   }
   .input-group input {
     padding: 10px;
     border: 1px solid #ccc;
     border-radius: 5px;
     width: 200px;
   }
   button {
     margin-top: 10px;
     padding: 10px 20px;
     background-color: #4caf50;
     color: white;
     border: none;
     border-radius: 5px;
     cursor: pointer;
   }
   ```

   **src/App.js**

   ```js
   import { useState } from "react";
   import "./App.css";

   function App() {
     const [inputText, setInputText] = useState("");
     const [outputText, setOutputText] = useState("");

     const handleSubmit = (e) => {
       e.preventDefault();
       fetch("/backend", {
         method: "POST",
         headers: { "Content-Type": "application/json" },
         body: JSON.stringify({ inputText }),
       })
         .then((res) => res.json())
         .then((data) => setOutputText(data.outputText))
         .catch(console.error);
     };

     return (
       <div className="app">
         <h2 className="title">Enter your message</h2>
         <form onSubmit={handleSubmit}>
           <div className="input-group">
             <label htmlFor="inputText">Request</label>
             <input
               id="inputText"
               type="text"
               value={inputText}
               onChange={(e) => setInputText(e.target.value)}
             />
           </div>
           <div className="input-group">
             <label htmlFor="outputText">Response</label>
             <input id="outputText" type="text" value={outputText} readOnly />
           </div>
           <button type="submit">Send</button>
         </form>
       </div>
     );
   }

   export default App;
   ```

   **src/index.js**

   ```js
   import React from "react";
   import ReactDOM from "react-dom/client";
   import App from "./App";

   const root = ReactDOM.createRoot(document.getElementById("root"));
   root.render(<App />);
   ```

2. Add a `.env` file at the project root to change the port and base URL:

   **.env**

   ```
   PORT=10001
   PUBLIC_URL=/frontend
   ```

3. Run `npm start` again and verify the app renders correctly:

   ![React app running](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-00-24.png)

### Build and Deploy

1. Build the production bundle:

   ```
   D:\react-spring\frontend>npm run build

   > frontend@0.1.0 build
   > react-scripts build

   Creating an optimized production build...
   Compiled successfully.
   ```

2. Install the `serve` static file server:

   ```
   D:\react-spring\frontend>npm install -g serve
   ```

3. Serve the build on port 10001:

   ```
   D:\react-spring\frontend>serve -s build -l 10001

         Serving!

         - Local:    http://localhost:10001
         - Network:  http://192.168.0.11:10001
   ```

4. Visit `http://localhost:10001` to confirm the production build is live:

   ![React production build](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-00-53.png)

## Spring

### Creating the Project

1. Install the **Spring Boot Extension Pack** in VS Code:

   ![Spring Boot Extension Pack](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-01-11.png)

2. Create a new Maven project via the command palette:

   ![Create Maven project](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-01-51.png)

3. Keep pressing Enter through the defaults, but change the Artifact ID from `demo` to `backend`:

   ![Set Artifact ID to backend](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-02-15.png)

4. Continue pressing Enter through JAR/WAR and Java version. For dependencies, select only these two:

   ![Select dependencies](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-02-29.png)

5. Press Enter on the folder selection screen:

   ![Folder selection](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-02-41.png)

6. Click **Open** in the bottom-right notification to open the project in a new window:

   ![Open project notification](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-02-58.png)

### Sample Code

1. The generated project structure:

   ![Project structure](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-03-17.png)

2. Only two files need to be modified:

   **TextController.java**

   ```java
   package com.example.backend.controller;

   import java.util.HashMap;
   import org.springframework.http.ResponseEntity;
   import org.springframework.web.bind.annotation.PostMapping;
   import org.springframework.web.bind.annotation.RequestBody;
   import org.springframework.web.bind.annotation.RestController;

   @RestController
   public class TextController {
       @PostMapping(value = "/", consumes = "application/json", produces = "application/json")
       public ResponseEntity<HashMap<String, String>> postText(
               @RequestBody HashMap<String, String> textMap) {
           System.out.println("React sent: " + textMap.get("inputText"));
           textMap.remove("inputText");
           textMap.put("outputText", "Spring replied: POST received");
           return ResponseEntity.ok().body(textMap);
       }
   }
   ```

   **src/main/resources/application.properties**

   ```
   spring.application.name=backend
   server.port=20001
   ```

3. Start the app and verify it runs:

   ```
   D:\react-spring\backend>mvnw spring-boot:run

   2024-08-19T12:51:05.765+09:00  INFO ... Started BackendApplication in 1.922 seconds
   ```

4. Test the API with Postman or the Thunder Client VS Code extension:

   ![API test with Postman](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-03-37.png)

5. The Spring console prints:

   ```
   React sent: does POST work?
   ```

### Build and Deploy

1. Stop the running Spring process with `Ctrl+C`:

   ```
   Terminate batch job (Y/N)? y
   ```

2. Build the JAR:

   ```
   D:\react-spring\backend>mvnw clean package

   [INFO] BUILD SUCCESS
   [INFO] Total time:  16.937 s
   ```

3. Run the built JAR:

   ```
   D:\react-spring\backend\target>java -jar backend-0.0.1-SNAPSHOT.jar

     .   ____          _            __ _ _
    /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
   ( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
    \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
     '  |____| .__|_| |_|_| |_\__, | / / / /
    =========|_|==============|___/=/_/_/_/

    :: Spring Boot ::                (v3.3.2)

   2024-08-19T12:58:46.696+09:00  INFO ... Started BackendApplication in 3.362 seconds
   ```

### Running React and Spring Together

1. Open a second console and start the built React app:

   ```
   D:\react-spring\frontend>serve -s build -l 10001

   Serving!
   - Local:    http://localhost:10001
   ```

2. Verify the request, response, and Spring console output all look correct:

   ![React + Spring working together](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-03-57.png)

## Network Configuration

### Firewall Rules

1. So far you can only access the app at `localhost`. Let's open it to the outside world.

2. First, check the server's IP address:

   ```
   C:\Users\user>ipconfig

   Ethernet adapter Ethernet:

      IPv4 Address. . . . . . . . . . . : 192.168.0.11
      Subnet Mask . . . . . . . . . . . : 255.255.255.0
      Default Gateway . . . . . . . . . : 192.168.0.1
   ```

3. From another device on the same network (connected via Wi-Fi or Ethernet), try opening `http://192.168.0.11/frontend`:

   ![Access attempt fails](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-04-19.png)

4. The connection fails because TCP port 80 is blocked by the Windows Firewall.

5. Open Windows Defender Firewall with Advanced Security:

   ![Open Windows Firewall](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-04-35.png)

6. Add a new **Inbound Rule** to allow HTTP (TCP 80) and HTTPS (TCP 443):

   ![New inbound rule step 1](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-04-48.png)
   ![New inbound rule step 2](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-04-54.png)
   ![New inbound rule step 3](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-04-59.png)
   ![New inbound rule step 4](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-05-05.png)
   ![New inbound rule step 5](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-05-10.png)

7. Refresh the page on the other device — it now loads:

   ![Page loads on other device](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-05-21.png)

### Assigning a Static IP to the Server

1. From the earlier `ipconfig` output, note the **Default Gateway** address (e.g. `192.168.0.1`).

2. Open that address in a browser to access your router's admin page. You can do this from any device on the network.

3. If you use an SK Broadband modem, the admin page looks like this. Instructions also follow for ipTIME routers:

   ![Router admin page](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-05-38.png)

4. Navigate to **Network → LAN → Static IP Assignment**:

   ![Static IP assignment menu](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-05-56.png)

5. Enter the server's MAC address and the static IP you want to assign:

   ![Static IP assignment form](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-06-15.png)

6. Find the MAC address with `ipconfig /all`:

   ```
   C:\Users\user>ipconfig /all

   Ethernet adapter Ethernet:

      Physical Address. . . . . . . . . : AA-BB-CC-DD-EE-FF
      IPv4 Address. . . . . . . . . . . : 192.168.0.11(Preferred)
   ```

7. On an ipTIME router, the setting is found here:

   ![ipTIME static IP menu](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-06-29.png)

### Port Forwarding

1. Navigate to **NAT → Port Forwarding** and add rules to forward TCP 80 and 443 to the server's static IP:

   ![Port forwarding rule](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-06-47.png)

2. On an ipTIME router:

   ![ipTIME port forwarding menu](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-07-00.png)

3. After saving the static IP and port forwarding settings, power-cycle the router:

   ![Rebooting router](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-07-17.png)

### Verifying External Access

1. Find the router's public IP address (shown in the router admin panel):

   ![Public IP address](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-07-38.png)

2. Turn off Wi-Fi on your phone and use mobile data. If the public IP is `203.0.113.42`, navigate to:
   `http://203.0.113.42/frontend`

   > **Note:** Devices on the same network cannot access the server via the public IP — use LTE/mobile data to test from outside.

   ![External access via mobile data](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-08-01.png)

## Connecting a Domain

### Purchasing a Domain on Gabia

1. The raw IP URL works but isn't very shareable. Let's attach a domain.

2. I searched for `example` on [Gabia](https://domain.gabia.com/) and bought a cheap domain:

   ![Gabia domain purchase](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-08-20.png)

3. Open the domain management page:

   ![Gabia domain management](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-08-35.png)

### Pointing the Domain to the Server

1. Click **Connect Domain**:

   ![Connect domain button](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-08-50.png)

2. Add two DNS A records pointing to your public IP:

   ![DNS A records](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-09-05.png)

3. DNS propagation is not instant. Mine took about 15 minutes.

4. Re-test from your phone on mobile data: `http://example.com/frontend`

   ![Domain working on mobile](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-09-25.png)

## HTTPS Setup

### SSL Certificates

1. A good SSL primer: [What is an SSL Certificate? — NordVPN](https://nordvpn.com/blog/what-is-ssl-certificate/)

   ![SSL certificate explanation](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-09-42.png)

2. Without HTTPS, every browser visit triggers this warning:

   ![Not secure warning](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-10-04.png)

3. Gabia sells SSL certificates for about ₩40,000/year. There's a free alternative.

   ![Gabia SSL pricing](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-10-29.png)

4. **Let's Encrypt** provides free SSL certificates:

   ![Let's Encrypt website](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-10-42.png)

5. On Windows, use [**win-acme**](https://www.win-acme.com/) — it handles both issuance and automatic renewal. Download the ZIP and extract it. I extracted to `C:\Program Files` and added it to PATH so `wacs.exe` is accessible from anywhere:

   ![win-acme in PATH](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-11-07.png)

### Issuing a Certificate with win-acme

1. Run `wacs.exe` and press `M` for full options:

   ```
   C:\Users\user>wacs

    A simple Windows ACMEv2 client (WACS)
    Software version 2.2.9.1701 (release, trimmed, standalone, 64-bit)
    Connecting to https://acme-v02.api.letsencrypt.org/...
    Connection OK!

    N: Create certificate (default settings)
    M: Create certificate (full options)
    ...
    Please choose from the menu: m
   ```

2. Select `2: Manual input`:

   ```
    1: Read bindings from IIS
    2: Manual input
    3: CSR created by another program
    C: Abort

    How shall we determine the domain(s) to include in the certificate?: 2
   ```

3. Enter your domain (comma-separated for www too):

   ```
    Host: example.com,www.example.com
   ```

4. Set a friendly name (I used the domain itself):

   ```
    Friendly name '[Manual] example.com'. <Enter> to accept or type desired name: example.com
   ```

5. Select `1: Separate certificate for each domain`:

   ```
    1: Separate certificate for each domain (e.g. *.example.com)
    2: Separate certificate for each host (e.g. sub.example.com)
    ...
    Would you like to split this source into multiple certificates?: 1
   ```

6. Select `1: [http] Save verification files on (network) path`:

   ```
    1: [http] Save verification files on (network) path
    2: [http] Serve verification files from memory
    ...
    How would you like prove ownership for the domain(s)?: 1
   ```

7. Enter the path to nginx's `html` folder:

   ```
    Path: C:\Users\user\nginx-1.26.2\html
   ```

   ![Verification file path](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-11-36.png)

8. `Copy default web.config?` → `no`

9. Select `2: RSA key`:

   ```
    1: Elliptic Curve key
    2: RSA key
    What kind of private key should be used?: 2
   ```

10. Enter the path where `.pem` files will be saved. Create a `cert` folder inside nginx's `conf` directory — nginx can only find files inside its own directory tree:

    ```
     File path: C:\Users\user\nginx-1.26.2\conf\cert
    ```

11. Private key password → `1: None`

12. Additional store → `5: No (additional) store steps`

13. Installation steps → `3: No (additional) installation steps`

14. Agree to the terms of service and enter your email for expiry notifications:

    ```
     Do you agree with the terms? (y*/n) - yes
     Enter email(s) for notifications: your@email.com
    ```

15. The `.pem` files are generated in the specified folder — certificate issuance succeeded:

    ```
     [example.com] Authorization result: valid
     [www.example.com] Authorization result: valid
     Downloading certificate example.com [example.com]
     Exporting .pem files to C:\Users\user\nginx-1.26.2\conf\cert
     Certificate example.com created
    ```

    ![Certificate files created](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-12-04.png)

### win-acme Notes

1. Let's Encrypt certificates expire after **90 days**.
2. As long as the PC stays on, win-acme renews them automatically.
3. Don't delete or move the generated `.pem` files — that breaks auto-renewal.
4. Open **Task Scheduler** to confirm a daily 09:00 task was created:

   ![Task Scheduler](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-12-19.png)
   ![Task Scheduler detail](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-12-24.png)

### Updating nginx to Use the Certificate

1. Now configure nginx to accept HTTPS (TCP 443) only:

   ![nginx.conf with SSL](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-12-37.png)

   **nginx.conf**

   ```nginx
   worker_processes 1;

   error_log logs/error.log;
   pid logs/nginx.pid;

   events {
     worker_connections 1024;
   }

   http {
     server {
       listen 443 ssl;
       server_name example.com www.example.com;
       charset utf-8;

       ssl_certificate cert/example.com-chain.pem;
       ssl_certificate_key cert/example.com-key.pem;

       ssl_session_cache shared:SSL:1m;
       ssl_session_timeout 5m;
       ssl_ciphers HIGH:!aNULL:!MD5;
       ssl_prefer_server_ciphers on;

       location / {
         root html;
         index index.html;
       }

       location /frontend {
         proxy_pass http://localhost:10001/;
       }

       location /backend {
         proxy_pass http://localhost:20001/;
       }
     }
   }
   ```

2. Reload nginx:

   ```
   C:\Users\user\nginx-1.26.2>nginx -s reload
   ```

3. Go back to the Windows Firewall inbound rules and remove the TCP 80 rule — only 443 should be open now:

   ![Remove TCP 80 firewall rule](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-12-54.png)

### Verifying HTTPS

1. Test from your phone on mobile data — the browser now shows the connection is secure:

   ![HTTPS confirmed on mobile](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-13-09.png)

## Deploying Multiple Projects Simultaneously

### Customizing nginx's index.html

1. Visiting the bare domain shows the default nginx page:

   ![Default nginx page](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-13-25.png)

2. I dressed it up using the [MVP.css](https://andybrewer.github.io/mvp/) classless CSS framework:

   ![Custom landing page](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-13-44.png)

   ```html
   <!DOCTYPE html>
   <html lang="en">
     <head>
       <meta charset="UTF-8" name="viewport" content="width=device-width, initial-scale=1.0" />
       <link rel="stylesheet" href="https://unpkg.com/mvp.css" />
       <style>a { text-decoration: none; }</style>
       <title>Your Name</title>
     </head>
     <body>
       <header>
         <nav>
           <a href="https://example.com"><h2>Your Name</h2></a>
           <ul>
             <li><a href="https://github.com/yourname" target="_blank">GitHub ↗</a></li>
           </ul>
         </nav>
         <h1>👋 Hello, I'm Your Name</h1>
         <p>Thanks for stopping by.</p>
         <br />
         <p>
           <a href="https://blog.example.com" target="_blank"><i>Blog</i></a>
           <a href="mailto:you@example.com" target="_blank"><b>Email</b></a>
         </p>
       </header>
       <main>
         <hr />
         <section>
           <header>
             <h2>Projects</h2>
             <p>A few things I've built.</p>
           </header>
           <aside>
             <h3>Project 1</h3>
             <p>A React + Spring side project, deployed from my laptop via nginx.</p>
             <p><a href="/project1" target="_blank"><em>Open</em></a></p>
           </aside>
           <aside>
             <h3>Project 2</h3>
             <p>Another React + Spring side project, also served from home.</p>
             <p><a href="/project2" target="_blank"><em>Open</em></a></p>
           </aside>
           <aside>
             <h3>Project 3</h3>
             <p>Third React + Spring side project, same setup.</p>
             <p><a href="/project3" target="_blank"><em>Open</em></a></p>
           </aside>
         </section>
       </main>
       <footer>
         <hr />
         <p>Made by <a href="https://github.com/yourname" target="_blank">Your Name ↗</a></p>
       </footer>
     </body>
   </html>
   ```

### Updating nginx Proxy Config for Multiple Projects

1. I cloned project 1 twice to get three projects running simultaneously with this URL scheme:

   ```
   # Project 1
   frontend: /project1  port 10001
   backend:  /api1      port 20001

   # Project 2
   frontend: /project2  port 10002
   backend:  /api2      port 20002

   # Project 3
   frontend: /project3  port 10003
   backend:  /api3      port 20003
   ```

2. Updated `nginx.conf` (the `http` block):

   ```nginx
   http {
     server {
       listen 443 ssl;
       server_name example.com www.example.com;
       charset utf-8;

       ssl_certificate cert/example.com-chain.pem;
       ssl_certificate_key cert/example.com-key.pem;

       ssl_session_cache shared:SSL:1m;
       ssl_session_timeout 5m;
       ssl_ciphers HIGH:!aNULL:!MD5;
       ssl_prefer_server_ciphers on;

       location / {
         root html;
         index index.html;
       }

       location /project1 { proxy_pass http://localhost:10001/; }
       location /api1      { proxy_pass http://localhost:20001/; }
       location /project2  { proxy_pass http://localhost:10002/; }
       location /api2      { proxy_pass http://localhost:20002/; }
       location /project3  { proxy_pass http://localhost:10003/; }
       location /api3      { proxy_pass http://localhost:20003/; }
     }
   }
   ```

### Updating React for Multiple Projects

**src/App.js** — change the fetch path:

```js
// Change /backend to /api1, /api2, or /api3
fetch("/api1", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ inputText }),
});
```

**.env** — change port and URL path per project:

```
PORT=10001
PUBLIC_URL=/project1
```

Rebuild after each change:

```
D:\react-spring\frontend>npm run build
```

### Updating Spring for Multiple Projects

**application.properties** — change the port per project:

```
spring.application.name=backend
server.port=20001
```

Rebuild:

```
D:\react-spring\backend>mvnw clean package
```

### Running Everything

1. After six builds, the files are ready:

   ![All build artifacts](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-14-13.png)

2. Open 6 console windows (CMD tabs in Windows 11 are a lifesaver) and start each process:

   ![6 consoles running](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-14-27.png)

## Final Result

![Final result demo](/images/deploying-react-spring-from-home-laptop/2024-11-24-21-14-53.gif)
