# Ethical Hacking Study Guide: Bind Shells, Reverse Shells, Apache, Firewalls, Nmap and Botnets

> **Lab safety:** Practise only on systems you own or have explicit
> permission to test. Bind and reverse shells can provide remote command
> execution. This guide explains their architecture and security
> implications and uses harmless Python message-exchange programs
> instead of remote command execution.

## Contents

1.  [Bind shell](#1-bind-shell)
2.  [Reverse shell](#2-reverse-shell)
3.  [Apache HTTP Server](#3-apache-http-server)
4.  [Firewalls](#4-firewalls)
5.  [Nmap](#5-nmap)
6.  [Botnets](#6-botnets)
7.  [How the topics connect](#7-how-all-these-topics-connect)
8.  [Practical learning roadmap](#8-practical-learning-roadmap)
9.  [Official documentation](#9-useful-official-documentation)
10. [Practical assignment](#10-practical-assignment)

------------------------------------------------------------------------

# 1. Bind shell

## 1.1 Definition

A **bind shell** is a remote-access mechanism in which a program runs on
a target machine, opens a listening network port, and waits for another
machine to connect.

The machine running the listening program is the server. The machine
initiating the connection is the client.

## 1.2 How it works

Imagine two machines:

-   Machine A: `192.168.1.110`
-   Machine B: `192.168.1.120`
-   Listening port: `6060`

The conceptual process is:

1.  Machine A creates a TCP socket.
2.  It binds the socket to an IP address and port.
3.  It listens for incoming connections.
4.  Machine B connects to that port.
5.  The connection is established.
6.  In a shell implementation, commands and their output would be
    exchanged over the connection.

## 1.3 Important Python socket methods

  Method              Purpose
  ------------------- ---------------------------------------------------
  `socket.socket()`   Creates a socket
  `bind()`            Associates the socket with an IP address and port
  `listen()`          Makes a TCP socket listen for connections
  `accept()`          Accepts an incoming connection
  `recv()`            Receives network data
  `sendall()`         Sends all supplied data
  `close()`           Closes the connection

## 1.4 Why is a bind shell useful to study in ethical hacking?

A penetration tester may study bind-shell behaviour to understand:

-   How exposed listening ports can create security risks.
-   Why unnecessary services should be disabled.
-   How firewalls restrict incoming connections.
-   How attackers may attempt to establish remote access.
-   How defenders identify suspicious listening processes.

**Security risk:** If a machine exposes an unauthenticated command
shell, an unauthorized person who can reach that port may gain control
of the machine.

## 1.5 Safe Python networking demonstration

This example teaches TCP socket concepts without executing
operating-system commands.

### `server.py`

``` python
import socket

HOST = "127.0.0.1"
PORT = 6060

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

server.setsockopt(
    socket.SOL_SOCKET,
    socket.SO_REUSEADDR,
    1
)

server.bind((HOST, PORT))
server.listen(1)

print(f"Listening on {HOST}:{PORT}")

try:
    conn, address = server.accept()

    with conn:
        print("Connected:", address)

        while True:
            data = conn.recv(1024)

            if not data:
                break

            message = data.decode("utf-8")
            print("Received:", message)

            conn.sendall(
                f"Server received: {message}".encode("utf-8")
            )

finally:
    server.close()
```

### `client.py`

``` python
import socket

HOST = "127.0.0.1"
PORT = 6060

client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

try:
    client.connect((HOST, PORT))

    while True:
        message = input("Message (q to quit): ")

        if message.lower() == "q":
            break

        client.sendall(message.encode("utf-8"))

        response = client.recv(1024)

        print(response.decode("utf-8"))

finally:
    client.close()
```

Run the server in one terminal:

``` powershell
python server.py
```

Run the client in another terminal:

``` powershell
python client.py
```

The server waits for a connection, and the client connects to it. This
demonstrates the network architecture without creating remote command
execution.

------------------------------------------------------------------------

# 2. Reverse shell

## 2.1 Definition

A **reverse shell** is a remote-access mechanism in which a program
running on a target machine initiates an outgoing connection to another
machine, often called the attacker's listener.

This reverses the connection direction of a bind shell.

## 2.2 How it works

Consider these two machines:

-   Target: `192.168.1.110`
-   Listener: `192.168.1.120`
-   Port: `6060`

Conceptually:

1.  The listener opens a TCP port.
2.  A program on the target initiates an outbound connection to the
    listener.
3.  The TCP connection is established.
4.  Data can travel in both directions.
5.  In a malicious shell implementation, this connection may be used to
    transmit commands and receive their output.

## 2.3 Bind shell vs reverse shell

  -----------------------------------------------------------------------
  Feature                 Bind shell              Reverse shell
  ----------------------- ----------------------- -----------------------
  Listening machine       Target                  Remote controller

  Connection initiator    Remote client           Target program

  Connection direction    Incoming to target      Outgoing from target

  Main network            Target's listening port Target must be able to
  requirement             must be reachable       make an outbound
                                                  connection

  Common defensive        Unexpected listening    Unexpected outbound
  concern                 ports                   connections
  -----------------------------------------------------------------------

## 2.4 Why is a reverse shell useful to study?

-   Understand how compromised machines may communicate with external
    controllers.
-   Identify suspicious outbound connections.
-   Test firewall egress policies.
-   Investigate malware behaviour.
-   Understand command-and-control communication.
-   Evaluate endpoint detection and response systems.

Some firewalls block unsolicited incoming connections but permit certain
outgoing connections. Therefore, an outbound connection can sometimes
cross a network boundary that would prevent a direct incoming
connection. Defenders must monitor both incoming and outgoing traffic.

## 2.5 Safe Python reverse-connection demonstration

This harmless version exchanges text messages. It does not execute
commands.

### `listener.py`

``` python
import socket

HOST = "127.0.0.1"
PORT = 6060

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind((HOST, PORT))
server.listen(1)

print("Waiting for an outgoing client connection...")

try:
    conn, address = server.accept()

    with conn:
        print("Connected:", address)

        while True:
            data = conn.recv(1024)

            if not data:
                break

            print("Client:", data.decode("utf-8"))

            conn.sendall(b"Message received")

finally:
    server.close()
```

### `callback_client.py`

``` python
import socket

LISTENER_HOST = "127.0.0.1"
PORT = 6060

client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

try:
    # The client initiates the connection.
    client.connect((LISTENER_HOST, PORT))

    message = "Hello from the connecting client"

    client.sendall(message.encode("utf-8"))

    response = client.recv(1024)

    print("Listener:", response.decode("utf-8"))

finally:
    client.close()
```

Run `listener.py` first, then `callback_client.py`.

-   In the bind example, the server waits for the client.
-   In this reverse-connection example, the client initiates a
    connection to the listener.

------------------------------------------------------------------------

# 3. Apache HTTP Server

## 3.1 What is Apache?

**Apache HTTP Server** is open-source web server software that receives
HTTP or HTTPS requests from clients, processes them according to its
configuration, and serves web content.

For example, if you create a website using HTML, CSS and JavaScript, a
web server can deliver those resources to a browser. Apache is one
option.

## 3.2 Why is Apache important in ethical hacking?

Apache is useful to study because it introduces:

1.  HTTP request and response handling.
2.  Web server configuration.
3.  Virtual hosts and domain hosting.
4.  Directory permissions.
5.  Access logs and error logs.
6.  TLS/HTTPS configuration.
7.  Security headers.
8.  Server-side vulnerabilities and misconfiguration.

An ethical hacker may investigate:

-   Directory listing accidentally enabled.
-   Outdated Apache versions.
-   Unnecessary modules.
-   Exposed administrative files.
-   Weak file permissions.
-   Incorrect virtual-host configuration.
-   Missing HTTPS configuration.

## 3.3 Installing Apache on Ubuntu/WSL

In your WSL Ubuntu terminal:

``` bash
sudo apt update
sudo apt install apache2 -y
```

Check its status:

``` bash
sudo systemctl status apache2
```

Start Apache:

``` bash
sudo systemctl start apache2
```

Enable automatic startup where supported:

``` bash
sudo systemctl enable apache2
```

Test it:

``` bash
curl -I http://127.0.0.1/
```

You should receive an HTTP response, commonly:

``` text
HTTP/1.1 200 OK
Server: Apache/...
Content-Type: text/html
```

You can also open this address in a browser:

``` text
http://127.0.0.1/
```

## 3.4 Apache's important directories

  Path                              Purpose
  --------------------------------- ------------------------------
  `/etc/apache2/`                   Main configuration directory
  `/etc/apache2/apache2.conf`       Main Apache configuration
  `/etc/apache2/ports.conf`         Listening ports
  `/etc/apache2/sites-available/`   Available virtual hosts
  `/etc/apache2/sites-enabled/`     Enabled virtual hosts
  `/etc/apache2/mods-available/`    Available modules
  `/etc/apache2/mods-enabled/`      Enabled modules
  `/var/www/html/`                  Default website directory
  `/var/log/apache2/access.log`     HTTP access log
  `/var/log/apache2/error.log`      Error log

## 3.5 Configure your first Apache website

Create a website directory:

``` bash
sudo mkdir -p /var/www/mywebsite
```

Create an HTML page:

``` bash
sudo nano /var/www/mywebsite/index.html
```

Add:

``` html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>My Apache Lab</title>
</head>
<body>
    <h1>Apache is working!</h1>
    <p>This is my ethical hacking laboratory.</p>
</body>
</html>
```

Create a virtual-host configuration:

``` bash
sudo nano /etc/apache2/sites-available/mywebsite.conf
```

Add:

``` apache
<VirtualHost *:80>

    ServerName mywebsite.local

    DocumentRoot /var/www/mywebsite

    <Directory /var/www/mywebsite>
        Options -Indexes
        AllowOverride None
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/mywebsite_error.log
    CustomLog ${APACHE_LOG_DIR}/mywebsite_access.log combined

</VirtualHost>
```

### Understand the configuration

  -----------------------------------------------------------------------
  Directive                           Meaning
  ----------------------------------- -----------------------------------
  `VirtualHost *:80`                  Handles HTTP traffic on port 80

  `ServerName`                        Website hostname

  `DocumentRoot`                      Website's document directory

  `Options -Indexes`                  Prevents automatic directory
                                      listing

  `AllowOverride None`                Prevents `.htaccess` overrides in
                                      this directory

  `Require all granted`               Allows visitors to access the
                                      directory

  `ErrorLog`                          Location of error records

  `CustomLog`                         Location of access records
  -----------------------------------------------------------------------

Enable the website:

``` bash
sudo a2ensite mywebsite.conf
```

Check configuration syntax:

``` bash
sudo apache2ctl configtest
```

Expected output:

``` text
Syntax OK
```

Reload Apache:

``` bash
sudo systemctl reload apache2
```

For local hostname testing, add an entry to the Windows hosts file using
an administrator editor:

``` text
127.0.0.1 mywebsite.local
```

Then test:

``` bash
curl -I http://mywebsite.local/
```

Inspect the logs:

``` bash
sudo tail -f /var/log/apache2/mywebsite_access.log
```

And:

``` bash
sudo tail -f /var/log/apache2/mywebsite_error.log
```

**Security lesson:** Logs help investigate unexpected requests, errors,
suspicious paths and repeated access attempts. They are important
evidence during incident response.

------------------------------------------------------------------------

# 4. Firewalls

## 4.1 What is a firewall?

A **firewall** is a security mechanism that monitors and controls
incoming and outgoing network traffic according to defined rules.

Think of it as a security guard at the entrance of a building. It
decides which network connections should be allowed and which should be
blocked.

## 4.2 Why is a firewall important?

Firewalls help protect systems against:

-   Unauthorized connections.
-   Unnecessary exposed ports.
-   Unauthorized remote access.
-   Suspicious outbound communication.
-   Some network-based attacks.
-   Unwanted access to web and database services.

For example, you might want HTTP and HTTPS to be accessible to website
visitors, but you should not expose your database to everyone on the
internet.

## 4.3 Important firewall concepts

  Concept             Meaning
  ------------------- ----------------------------------------
  Inbound             Traffic entering your machine
  Outbound            Traffic leaving your machine
  Allow               Permit matching traffic
  Deny                Block matching traffic
  Port                Identifies a network service
  Protocol            TCP, UDP, ICMP, etc.
  Source IP           Where traffic originates
  Destination IP      Where traffic is going
  Stateful firewall   Tracks connection states
  Rule priority       Determines which rule takes precedence

## 4.4 Configure Windows Defender Firewall

On Windows 11, open PowerShell as Administrator.

Check firewall status:

``` powershell
Get-NetFirewallProfile
```

Enable all three firewall profiles:

``` powershell
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled True
```

The profiles mean:

-   `Domain`: Network connected to an organization domain.
-   `Private`: Trusted private network.
-   `Public`: Untrusted networks, such as public Wi-Fi.

Check again:

``` powershell
Get-NetFirewallProfile |
    Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
```

### Create an inbound rule for Apache HTTP

If you are running Apache on Windows and want to permit HTTP traffic:

``` powershell
New-NetFirewallRule `
    -DisplayName "Allow Apache HTTP" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 80 `
    -Action Allow `
    -Profile Private
```

This allows TCP port 80 on the Private profile. Do not create
public-facing rules unless you actually need them.

### Remove the rule

``` powershell
Remove-NetFirewallRule -DisplayName "Allow Apache HTTP"
```

### List firewall rules

``` powershell
Get-NetFirewallRule
```

Filter by name:

``` powershell
Get-NetFirewallRule -DisplayName "*Apache*"
```

## 4.5 Configure Ubuntu firewall using UFW

UFW means **Uncomplicated Firewall**. It provides a simpler interface to
Ubuntu's firewall framework.

Install it if necessary:

``` bash
sudo apt update
sudo apt install ufw -y
```

Check its status:

``` bash
sudo ufw status verbose
```

If you administer the machine remotely over SSH, allow SSH before
enabling UFW:

``` bash
sudo ufw allow 22/tcp
```

For an Apache website:

``` bash
sudo ufw allow 80/tcp
```

For HTTPS:

``` bash
sudo ufw allow 443/tcp
```

Set a default policy:

``` bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Enable the firewall:

``` bash
sudo ufw enable
```

Check:

``` bash
sudo ufw status numbered
```

Delete a rule using its number:

``` bash
sudo ufw status numbered
sudo ufw delete 2
```

**Caution:** Enabling UFW with an SSH connection can lock you out if you
have not allowed SSH first. In WSL, firewall behaviour can also differ
from a normal standalone Ubuntu server because Windows manages
networking around the WSL virtual environment.

## 4.6 How to test a firewall

Testing should establish whether a service is reachable from the
intended source.

First, check whether Apache is running:

``` bash
sudo systemctl status apache2
```

Check whether port 80 is listening:

``` bash
sudo ss -lntp
```

Test locally:

``` bash
curl -I http://127.0.0.1/
```

Check the firewall:

``` bash
sudo ufw status verbose
```

Then, from a separate machine on your authorized lab network, test the
service's reachability. A service can be listening while a firewall
blocks remote access. Distinguish local service status from remote
network reachability.

------------------------------------------------------------------------

# 5. Nmap

## 5.1 What is Nmap?

**Nmap** means **Network Mapper**. It is a network discovery and
security auditing tool used to identify:

-   Active hosts.
-   Open ports.
-   Closed ports.
-   Filtered ports.
-   Running services.
-   Service versions.
-   Some operating-system characteristics.

## 5.2 Why is Nmap important in ethical hacking?

Imagine you are hired to audit a company's web server. You need to
discover:

1.  Is the server reachable?
2.  Which ports are open?
3.  Which services are running?
4.  Is SSH exposed?
5.  Is Apache running?
6.  Is a database exposed unnecessarily?
7.  Are there unexpected services?

Nmap helps you answer these questions.

## 5.3 Install Nmap

On Ubuntu:

``` bash
sudo apt update
sudo apt install nmap -y
```

Check the version:

``` bash
nmap --version
```

On Windows, download Nmap from its official website:

<https://nmap.org/download.html>

## 5.4 Important Nmap commands

Only scan your own machine or systems for which you have authorization.

### 1. Scan localhost

``` bash
nmap 127.0.0.1
```

Scans the most common TCP ports by default.

### 2. Scan selected ports

``` bash
nmap -p 22,80,443,6060 127.0.0.1
```

`-p` specifies the ports to scan.

### 3. Scan a port range

``` bash
nmap -p 1-1000 127.0.0.1
```

### 4. Detect service versions

``` bash
nmap -sV -p 80,443 127.0.0.1
```

`-sV` attempts to identify the service and its version.

### 5. Discover hosts without a port scan

``` bash
nmap -sn 127.0.0.1
```

`-sn` performs host discovery without the usual port scan.

### 6. Save scan results

``` bash
nmap -oN scan-results.txt 127.0.0.1
```

`-oN` saves normal output to a file.

## 5.5 Understanding Nmap results

Example:

``` text
PORT     STATE     SERVICE

22/tcp   closed    ssh
80/tcp   open      http
443/tcp  closed    https
6060/tcp filtered  unknown
```

  -----------------------------------------------------------------------
  State                               Meaning
  ----------------------------------- -----------------------------------
  `open`                              An application is accepting
                                      connections

  `closed`                            The host responds, but no service
                                      is listening

  `filtered`                          A firewall or network filter
                                      prevents Nmap from determining the
                                      port state

  `unfiltered`                        The port is reachable, but its
                                      open/closed state is not determined

  `open|filtered`                     Nmap cannot distinguish an open
                                      port from a filtered one
  -----------------------------------------------------------------------

These states describe what Nmap can determine from its scan, not
necessarily the complete state of the service.

## 5.6 Nmap and firewalls: a lab exercise

1.  Start Apache.
2.  Scan port 80 on your lab machine.
3.  Configure the firewall to block the relevant incoming connection.
4.  Scan again from a separate machine on your authorized lab network.
5.  Compare the results.

You may see `open` before the firewall rule and `filtered` after it,
depending on the firewall response and network path.

Official reference: <https://nmap.org/docs.html>

------------------------------------------------------------------------

# 6. Botnets

## 6.1 What is a botnet?

A **botnet** is a group of compromised computers, servers, routers, IoT
devices or other connected systems that are controlled or coordinated by
an operator.

Each compromised device is commonly called a **bot** or **zombie**.

The operator may use a command-and-control (C2) mechanism to communicate
with the compromised devices or coordinate activity.

## 6.2 How does a botnet work?

A simplified lifecycle:

1.  **Initial infection:** A device becomes compromised.
2.  **Persistence:** Malicious software attempts to remain active.
3.  **Communication:** The infected device communicates with its
    controller or peers.
4.  **Coordination:** Devices receive instructions.
5.  **Action:** The botnet carries out malicious activity.

## 6.3 What are botnets used for?

  -----------------------------------------------------------------------
  Malicious activity                  Explanation
  ----------------------------------- -----------------------------------
  DDoS attacks                        Flooding a service with traffic to
                                      make it unavailable

  Spam                                Sending large volumes of unwanted
                                      messages

  Credential theft                    Stealing login information

  Malware distribution                Delivering additional malicious
                                      software

  Cryptomining                        Using victims' computing resources
                                      to mine cryptocurrency

  Proxy abuse                         Using infected devices to relay
                                      traffic

  Data theft                          Collecting information from
                                      compromised systems
  -----------------------------------------------------------------------

## 6.4 What is a C2 server?

C2 means **Command and Control**. It is infrastructure used to
coordinate compromised systems.

For example, a botnet controller may communicate with infected devices
through:

-   HTTP or HTTPS.
-   DNS.
-   IRC (historically used by some botnets).
-   Custom TCP or UDP protocols.
-   Peer-to-peer communication.

A defender should investigate unusual periodic connections, unexpected
DNS requests, unknown processes and unexplained outbound traffic.

## 6.5 Why should ethical hackers study botnets?

Understanding botnets helps you:

-   Identify infected devices.
-   Recognize suspicious network traffic.
-   Investigate malware.
-   Understand DDoS attacks.
-   Build network monitoring systems.
-   Improve firewall and endpoint security.
-   Develop incident-response procedures.

A safe learning project is a local simulator that exchanges harmless
status messages between Python clients and a controller. Do not build
software that infects devices, hides malware, spreads to other machines
or coordinates attacks.

## 6.6 How can you detect a possible bot infection?

On Linux, inspect listening services:

``` bash
sudo ss -tulpn
```

Inspect network connections:

``` bash
ss -tunap
```

Inspect running processes:

``` bash
ps aux
```

On Windows, inspect connections:

``` powershell
Get-NetTCPConnection
```

Find processes associated with connections:

``` powershell
Get-NetTCPConnection |
    Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State, OwningProcess
```

Inspect processes:

``` powershell
Get-Process
```

These commands help investigate suspicious behaviour. An unfamiliar
connection or process is not automatically proof of a botnet infection.

------------------------------------------------------------------------

# 7. How all these topics connect

These topics are not isolated. They are connected through network
security.

  -----------------------------------------------------------------------
  Technology                          Role in network security
  ----------------------------------- -----------------------------------
  Apache                              Provides web services on HTTP/HTTPS
                                      ports

  Firewall                            Controls which connections can
                                      reach or leave a server

  Nmap                                Helps audit reachable hosts, ports
                                      and services

  Bind shell concepts                 Help explain remote-access risks
                                      and exposed listening ports

  Reverse shell concepts              Help explain suspicious outbound
                                      connections

  Botnet investigation                Helps identify compromised devices
                                      and command-and-control
                                      communication
  -----------------------------------------------------------------------

A simplified workflow:

1.  Apache provides a web service.
2.  A firewall controls access to that service.
3.  Nmap helps audit which ports and services are reachable.
4.  Bind and reverse shell concepts help explain remote-access risks.
5.  Network monitoring helps investigate suspicious communications,
    including possible botnet activity.

------------------------------------------------------------------------

# 8. Practical learning roadmap

  -----------------------------------------------------------------------
  Stage                   Practical project       What you should learn
  ----------------------- ----------------------- -----------------------
  1                       Python TCP              `socket`, `bind`,
                          client/server           `listen`, `accept`,
                                                  `connect`, `sendall`,
                                                  `recv`

  2                       Bind/reverse connection Connection direction
                          demonstrations          and network
                                                  architecture

  3                       Apache installation     Web server
                                                  configuration and HTTP

  4                       Apache access-log       Python `re`, `pathlib`,
                          analyser                `collections`,
                                                  `logging`

  5                       Firewall configuration  Inbound/outbound rules
                                                  and port restrictions

  6                       Nmap lab                Host discovery, port
                                                  scanning and service
                                                  detection

  7                       Python network monitor  Detect new or
                                                  unexpected connections

  8                       Botnet traffic          Understand C2
                          investigation           communication and
                                                  suspicious traffic
  -----------------------------------------------------------------------

## Suggested order

Do not rush into botnets or advanced exploitation. First become
comfortable with:

1.  TCP sockets and networking fundamentals.
2.  Apache and HTTP.
3.  Firewall rules.
4.  Nmap and port states.
5.  Network monitoring and log analysis.
6.  Malware and botnet investigation.

These foundations will make the advanced topics easier to understand.

------------------------------------------------------------------------

# 9. Useful official documentation

-   **Apache HTTP Server:** <https://httpd.apache.org/docs/>
-   **Nmap reference guide:** <https://nmap.org/docs.html>
-   **Ubuntu firewall documentation:**
    <https://documentation.ubuntu.com/server/how-to/security/firewalls/>
-   **Windows Defender Firewall documentation:**
    <https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/>
-   **CISA cybersecurity guidance:**
    <https://www.cisa.gov/stopransomware>

------------------------------------------------------------------------

# 10. Practical assignment

Use your WSL Ubuntu environment and Windows PowerShell.

## Lab checklist

-   [ ] Install Apache on WSL Ubuntu.
-   [ ] Create an HTML page and serve it with Apache.
-   [ ] Inspect Apache's listening port using `ss`.
-   [ ] Check the Apache access and error logs.
-   [ ] Install Nmap and scan `127.0.0.1`.
-   [ ] Configure UFW to allow HTTP.
-   [ ] Compare Nmap results before and after the firewall change.
-   [ ] Run the Python TCP message server and client.
-   [ ] Explain the difference between bind and reverse connections.
-   [ ] Write down three ways to detect suspicious outbound connections.

## Questions to test your understanding

1.  What is the main difference between a bind shell and a reverse
    shell?
2.  Why might an outbound connection work when an inbound connection is
    blocked?
3.  What does `bind()` do in Python socket programming?
4.  What is the difference between `listen()` and `accept()`?
5.  What is Apache's `DocumentRoot`?
6.  Why should directory listing often be disabled?
7.  What is the difference between inbound and outbound firewall rules?
8.  What does `filtered` mean in an Nmap scan?
9.  Why is an open port not automatically a vulnerability?
10. What is a botnet C2 server?
11. Why should defenders monitor outbound network connections?
12. How can Apache logs help during an investigation?

------------------------------------------------------------------------

**Final recommendation:** Build a Python Network Security Monitoring
Tool that checks listening ports, records network connections, analyses
Apache logs and reports suspicious activity. This will bring together
Python, Linux, networking, firewalls, web servers and ethical hacking in
one coherent project.
