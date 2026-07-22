Project 8: Load Balancer Solution With Apache (L7 Application Load Balancing)

## 📌 Project Overview
In this project, we enhanced the **DevOps Tooling Website Solution** deployed in Project 7 by implementing an **Apache Layer 7 (Application) Load Balancer** on an Ubuntu EC2 instance (`Project-8-apache-lb`). 

As web traffic grows, relying on single IP addresses or asking users to switch between multiple server addresses leads to a poor user experience and creates single points of failure. By placing a **Load Balancer (LB)** in front of our Web Servers, we achieve:
* **Single Point of Entry:** Users access the application via a single URL/IP.
* **Horizontal Scalability:** Effortlessly add or remove web servers to handle varying traffic loads.
* **Optimal Traffic Distribution:** Load is balanced across healthy backend nodes based on routing algorithms (e.g., `bytraffic`, `byrequests`).
* **High Availability & Fault Tolerance:** Failure of an individual backend web server does not cause a total service outage.

---

## 🏗️ Architecture & Infrastructure Breakdown

The updated multi-tier architecture consists of:
1. **Client Tier:** End-user browser sending HTTP requests on TCP Port 80.
2. **Load Balancer Tier:** Apache L7 Load Balancer (`Project-8-apache-lb` on Ubuntu 20.04) distributing incoming requests.
3. **Web Tier:** 2x RHEL8 Web Servers running Apache (`httpd`) and PHP (`Web-Server 1` & `Web-Server 2`).
4. **Data Tier:** 
   * **MySQL DB Server** (Ubuntu 20.04) on TCP Port 3306.
   * **NFS Server** (RHEL8) exporting shared storage (`/mnt/apps`) mounted to `/var/www` across web servers.
  
       ┌────────────────────────┐
       │   [ Client Browser ]   │
       └───────────┬────────────┘
                   │
            HTTP (TCP Port 80)
                   │
                   ▼
  ┌─────────────────────────────────┐
  │   Apache Layer 7 Load Balancer  │
  │     ( Project-8-apache-lb )     │
  └────────────────┬────────────────┘
                   │
         ┌─────────┴─────────┐
  HTTP   │                   │   HTTP
 (Port 80)                   (Port 80)
         │                   │
         ▼                   ▼
┌──────────────────┐  ┌──────────────────┐
│   Web Server 1   │  │   Web Server 2   │
│ (RHEL8 / Apache) │  │ (RHEL8 / Apache) │
└────────┬─────────┘  └────────┬─────────┘
         │                     │
         │   NFS (2049 / 111)  │
         ├─────────────────────┤
         │                     │
         ▼                     ▼
┌──────────────────┐  ┌──────────────────┐
│  MySQL DB Server │  │    NFS Server    │
│  (Ubuntu 20.04)  │  │     (RHEL8)      │
│  TCP Port 3306   │  │ /var/www -> NFS  │
└──────────────────┘  └──────────────────┘

__________________________________________

⚙️ Step-by-Step Implementation Guide

Step 1: EC2 Provisioning & Security Group Setup

1-Launch an Ubuntu Server 20.04 LTS instance named Project-8-apache-lb.  
2-Ensure all 5 instances in the cluster are running smoothly:  

Project 7 -NFS  
Project 7 -web server 1  
Project7 - DB  
Project 7 -Web server 2  
Project-8-apache-lb  

3-Configure the Security Group for Project-8-apache-lb by opening Inbound TCP Port 80 to 0.0.0.0/0.

Step 2: Install Apache and Required Modules

SSH into the Project-8-apache-lb instance and install Apache along with proxy modules

# Update package repositories and install Apache2
sudo apt update -y
sudo apt install apache2 -y
sudo apt-get install libxml2-dev -y

# Enable required Apache proxy modules
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic

# Restart Apache to apply module changes
sudo systemctl restart apache2
sudo systemctl status apache2

Step 3: Configure Local DNS Name Resolution (/etc/hosts)

To simplify backend management and avoid hardcoding private IP addresses into VirtualHost configurations, set up local DNS mapping on the Load Balancer

sudo vi /etc/hosts

Add the private IP addresses of your web servers with custom local aliases

172.31.23.26  Web1
172.31.23.88  Web2

Test Local DNS Connectivity:
Verify that the LB can communicate with both web servers using their local names

curl -I http://Web1
curl -I http://Web2

Both requests return HTTP/1.1 302 Found, confirming local resolution and HTTP reachability.

Step 4: Configure Apache Load Balancer Routing
Edit the default VirtualHost configuration file

sudo nano /etc/apache2/sites-available/000-default.conf

Insert the <Proxy> cluster configuration inside <VirtualHost *:80>

<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    <Proxy "balancer://mycluster">
        BalancerMember http://Web1:80 loadfactor=5 timeout=1
        BalancerMember http://Web2:80 loadfactor=5 timeout=1
        ProxySet lbmethod=bytraffic
    </Proxy>

    ProxyPreserveHost On
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

Test Configuration Syntax & Restart Service:

# Verify configuration syntax
sudo apache2ctl configtest

# Restart Apache
sudo systemctl restart apache2


Step 5: Verify Load Balancing & Web Output
1. Verify Full HTML Content Retrieval
Execute curl from the Load Balancer to inspect the raw HTML payload served by each backend node

Web1 HTML Response:

Web2 HTML Response:

2. Access via Web Browser
Open a browser and navigate to the Load Balancer's Public IP address
http://<Load-Balancer-Public-IP>/login.php

The StegTechHub Tooling Login interface loads seamlessly

3. Real-time Access Log Verification
To verify that traffic is actively balanced across both servers, open SSH sessions to Web Server 1 and Web Server 2 and tail their access logs

# Run on Web Server 1 & Web Server 2
sudo tail -f /var/log/httpd/access_log

Refresh the browser multiple times. Observe GET /index.php requests arriving at both nodes from the Load Balancer's private IP (172.31.23.224)[cite: 1]:

Web Server 1 Access Log:

Web Server 2 Access Log:

## 🛠️ Key Workarounds & Troubleshooting Fixes

| Issue Encountered | Root Cause | Workaround / Solution Applied |
| :--- | :--- | :--- |
| **HTTP 404 / Log Locking Across Web Servers** | In Project 7, `/var/log/httpd/` was mistakenly mounted to NFS, causing both web servers to overwrite and lock log files. | Unmounted `/var/log/httpd/` from NFS on both web servers so each node maintains independent local access logs (`/var/log/httpd/access_log`). |
| **`BalancerMember` Directive Error in Sub-Location** | Placing `BalancerMember` inside `<Location>` blocks caused syntax validation errors (`apache2ctl configtest`). | Defined the `<Proxy "balancer://mycluster">` block globally inside `<VirtualHost *:80>` rather than nested inside location blocks. |
| **IP Management Overhead** | Hardcoding private IPs in Apache configuration makes scaling or replacing instances tedious. | Configured local DNS name mapping inside `/etc/hosts` on the Load Balancer (`Web1`, `Web2`). |
| **Session Loss Across Refresh (Sticky Sessions)** | Round-robin / traffic balancing redirects users to different servers, causing session termination if sessions aren't synchronized. | Leveraged shared NFS storage for PHP sessions and/or database session persistence across web nodes. |



Key Takeaways & Advanced Concepts
1. Load Balancing Algorithms in Apache (mod_proxy_balancer)
bytraffic: Balances incoming load based on total network bandwidth/traffic volume (used in this project).

byrequests: Distributes requests evenly across backend nodes in a round-robin style based on request counts.

bybusyness: Routes traffic to the server with the fewest active pending requests.

2. Sticky Sessions (Session Affinity)
Sticky sessions ensure that a user's requests during a session are consistently routed to the same backend web server. This is essential for non-stateless web applications that store session state locally rather than in a distributed cache like Redis or Memcached.

🎉 Conclusion
The Apache Layer 7 Load Balancer successfully distributes incoming HTTP requests across Web1 and Web2. The setup provides high availability, fault tolerance, and invisible infrastructure complexity for end users accessing StegTechHub Tooling.
