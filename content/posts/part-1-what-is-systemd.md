---
title: "Part 1 - What Is Systemd"
date: 2023-11-28T22:42:48+01:00
draft: false
tags: 18min server systemd
---

# What is systemd?
systemd is a deamon which handles the steps after the kernel is loaded with it's important modules.

So you can say it's an initialization system. That's another definition I found in the internet.

## Units
systemd is managed by units. There are different types of units:
- Service Units
- Socket Units
- Target Units
- Device Units
- Mount Units
- Automount Units
- Snapshot Units
- Timer Units
- Swap Units
- Path Units
- Slice Units
- Scope Units

I think the most interesting for me are the Service Units.

A unit is basically a textfile with a definition what the unit should do.

## systemctl
That's the command to control the services. You can `start`, `stop`, `reload` and `restart` services or get 
information about the current `status`.

You can also `enable` and `disable` a service. When a service is enabled, it will automatically start after the
server is started.

# Create a new service
It's easy to create an own service unit. You just need to create a new file with the `.service` ending and put
it into the `/etc/systemd/system` directory. How does this service file should look like? What options are available?

## Basic example
Example:
```
[Unit]
Description=My Dashboard

[Service]
Type=simple
ExecStart=python3 /home/myl3z/code/mydashboard.py

[Install]
WantedBy=multi-user.target
```

In this example I have a small python script `mydashboard.py` in the folder `/home/myl3z/code/` which is executable. It can do 
anything, for example running a webserver with a website, maybe something like a little dashboard for myself.

## Basic python3 webserver
I have copied a very basic example of a webserver which could represent a dashboard:

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import time

hostName = "localhost"
serverPort = 8080

class MyServer(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.end_headers()
        self.wfile.write(bytes("<html><head><title>My Dashboard</title></head>", "utf-8"))
        self.wfile.write(bytes("<p>Request: %s</p>" % self.path, "utf-8"))
        self.wfile.write(bytes("<body>", "utf-8"))
        self.wfile.write(bytes("<p>This is an dashboard.</p>", "utf-8"))
        self.wfile.write(bytes("</body></html>", "utf-8"))

if __name__ == "__main__":        
    webServer = HTTPServer((hostName, serverPort), MyServer)
    print("Server started http://%s:%s" % (hostName, serverPort))

    try:
        webServer.serve_forever()
    except KeyboardInterrupt:
        pass

    webServer.server_close()
    print("Server stopped.")
```

If I run this with `python3 mydashboard.py` I can request it via the browser with http://localhost:8080/hi and the response will be 'hi'.

But I have to run the python script manually everytime. So I will put it in a service and enable it, so it's online everytime my machine is started!

SOUNDS GREAT!

## Put it in a service
The service file will be `mydashboard.service` and I put it into `/etc/systemd/system`.

Now I can check the `status` with `systemctl status mydashboard`, which will return:
### Status
```
○ mydashboard.service - My Dashboard
     Loaded: loaded (/etc/systemd/system/mydashboard.service; disabled; vendor preset: enabled)
     Active: inactive (dead)
```

### Start
The system recognize my dashboard service. Now I can start it to see if it's working. Start with the command: `systemctl start mydashboard`. There 
will be no response to that, but a new status request with `systemctl status mydashboard` gives me the output:

```
● mydashboard.service - My Dashboard
     Loaded: loaded (/etc/systemd/system/mydashboard.service; disabled; vendor preset: enabled)
     Active: active (running) since Wed 2023-11-29 22:11:04 CET; 2s ago
   Main PID: 9365 (python3)
      Tasks: 1 (limit: 2184)
     Memory: 8.1M
        CPU: 27ms
     CGroup: /system.slice/mydashboard.service
             └─9365 python3 /home/parallels/code/mydashboard.py

Nov 29 22:11:04 ubuntu-linux-22-04-02-desktop systemd[1]: Started My Dashboard.
```

IT'S ALIVE! Muhahahaha.

### Stop
And to complete it we can stop it with `systemctl stop mydashboard`.

If we run the status command again, we will get more output, because the logs are shown: 

```
 mydashboard.service - My Dashboard
     Loaded: loaded (/etc/systemd/system/mydashboard.service; disabled; vendor preset: enabled)
     Active: inactive (dead)

Nov 29 22:11:04 ubuntu-linux-22-04-02-desktop systemd[1]: Started My Dashboard.
Nov 29 22:12:24 ubuntu-linux-22-04-02-desktop python3[9365]: 127.0.0.1 - - [29/Nov/2023 22:12:24] "GET /hi HTTP/1.1
" 200 -
Nov 29 22:12:24 ubuntu-linux-22-04-02-desktop python3[9365]: 127.0.0.1 - - [29/Nov/2023 22:12:24] "GET /hi HTTP/1.1
" 200 -
Nov 29 22:12:25 ubuntu-linux-22-04-02-desktop python3[9365]: 127.0.0.1 - - [29/Nov/2023 22:12:25] "GET /hi2 HTTP/1.
1" 200 -
Nov 29 22:12:27 ubuntu-linux-22-04-02-desktop python3[9365]: 127.0.0.1 - - [29/Nov/2023 22:12:27] "GET /hi HTTP/1.1
" 200 -
Nov 29 22:12:59 ubuntu-linux-22-04-02-desktop systemd[1]: Stopping My Dashboard...
Nov 29 22:12:59 ubuntu-linux-22-04-02-desktop systemd[1]: mydashboard.service: Deactivated successfully.
Nov 29 22:12:59 ubuntu-linux-22-04-02-desktop systemd[1]: Stopped My Dashboard.
```

That are some requests I generated by opening the page in the browser.

### Enable
At last step I will enable it, so it will be working after reboot. Easy with `systemctl enable mydashboard`.

After the reboot I just try to access the dashboard in the browser. And it's working. Feels good.

# Part 2 is following
I've learned how to create a simple service, start, stop and enable it. So this was the first part, a second part
will follow!