# ⚡ Apache Load Balancer Configuration for Tooling Website

This project demonstrates how to deploy and configure an Apache Load Balancer on an Ubuntu EC2 instance to distribute traffic across multiple RHEL8 web servers. The load balancer ensures that users accessing the application are served seamlessly by any of the backend web servers.

---
## ⚡ Understanding Load Balancers: Types, Benefits, and Why They Are Essential in This Architecture  

Modern web applications must serve **many users simultaneously**, provide **high availability**, and maintain **consistent performance**. A single server is often not enough to handle these demands. This is where **Load Balancers (LBs)** come in.  

---

## 🔎 What is a Load Balancer?  

A **Load Balancer** is a system (software or hardware) that distributes incoming network traffic across multiple backend servers. By spreading requests intelligently, the LB ensures:  

- **No single server is overwhelmed** with requests.  
- **High availability**, since traffic can be redirected if one server fails.  
- **Scalability**, as more servers can be added behind the LB.  
- **Improved user experience** through faster response times.  

Think of it like a **traffic officer at a busy intersection**, directing cars (requests) to different roads (servers) to prevent jams.  

---

## 🛠️ Types of Load Balancers  

Load balancers can be categorized based on **how they operate**:  

### 1. Layer 4 Load Balancers (Transport Layer)  
- Operate at the **TCP/UDP** level.  
- Make decisions based on **IP address and port**.  
- Very fast, but limited in flexibility.  
- Example: *AWS Network Load Balancer (NLB)*.  

### 2. Layer 7 Load Balancers (Application Layer)  
- Operate at the **HTTP/HTTPS** level.  
- Make routing decisions based on **application data**, like URLs, cookies, or headers.  
- Can perform content-based routing (e.g., send image requests to one server group, API calls to another).  
- Example: *Apache HTTP Server, Nginx, AWS Application Load Balancer (ALB)*.  

### 3. Hardware Load Balancers  
- Dedicated physical appliances.  
- Offer very high performance but are expensive and less flexible.  

### 4. Software Load Balancers  
- Run on general-purpose servers.  
- Cost-effective, flexible, and widely used in cloud environments.  
- Example: *Apache, Nginx, HAProxy*.  

---

## ⚙️ Load Balancing Algorithms  

Load balancers use algorithms to decide how traffic is distributed:  

- **Round Robin** → Each request is sent to servers in sequence.  
- **Least Connections** → Sends traffic to the server with the fewest active connections.  
- **By Traffic (`lbmethod_bytraffic`)** → Distributes load based on the amount of traffic.  
- **By Requests** → Distributes evenly by request count.  
- **By Busyness** → Prioritizes servers with the lowest CPU/IO load.  

---

## ✅ Why a Load Balancer is Essential in This Architecture  

In this project setup, we have:  

- **3 RHEL8 Web Servers** → Serving the application content.  
- **1 MySQL Database (Ubuntu)** → Central data store.  
- **1 NFS Server (RHEL8)** → Shared storage for consistency.  

The **Apache Load Balancer (Ubuntu 20.04)** plays a critical role:  

1. **High Availability**  
   - If one web server fails, the LB automatically routes traffic to healthy servers.  
   - This ensures zero downtime for users.  

2. **Scalability**  
   - We can add or remove web servers behind the LB without affecting the frontend access.  
   - The system can grow as demand increases.  

3. **Performance Optimization**  
   - Traffic is intelligently distributed to prevent overloading a single server.  
   - This results in faster response times and better user experience.  

4. **Centralized Access Point**  
   - Users don’t need to know individual server IPs.  
   - The LB provides a **single public endpoint**, simplifying management.  

5. **Flexibility with Load Balancing Methods**  
   - By configuring `lbmethod_bytraffic`, we ensure distribution is proportional to server traffic load.  
   - We could switch to **byrequests** or **bybusyness** depending on performance needs.  

---

## 🌐 Real-World Analogy  

Picture a busy intersection. If cars (requests) rush in without control, one road gets jammed while others are empty. A traffic light (load balancer) directs cars in turns, preventing congestion and ensuring smooth traffic flow across all roads.  



A **Load Balancer is not just a tool, but a core component of resilient system design**. It ensures that our web architecture is:  

- **Reliable** (fault-tolerant)  
- **Scalable** (ready for growth)  
- **Efficient** (optimized for performance)  

By deploying **Apache as a Load Balancer**, we’ve transformed a collection of web servers which we deployed in our earlier project [DevOps Tooling Website Solution](https://github.com/princemaxi/DevOps-Tooling-Website) into a **robust, production-ready architecture** capable of handling real-world traffic demands.  

---

# 📋 Architecture Overview

- **3 RHEL8 Web Servers (serving the application)**
- **1 Ubuntu 20.04 MySQL Database Server**
- **1 RHEL8 NFS Server**
- **1 Ubuntu 20.04 Apache Load Balancer (LB)**

![alt image](/images/z.png)
---

# 🚀 Steps to Configure Apache Load Balancer
## 1. Launch the Load Balancer Instance

- Create an Ubuntu 20.04 EC2 instance named: `project-8-apache-lb`
![alt image](/images/1.png)
- Update security group: allow inbound TCP port 80.
- SSH into the instance
![alt image](/images/2.png)

## 2. Install Apache and Required Modules
```bash
# Update package index
sudo apt update  

# Install Apache2
sudo apt install apache2 -y  

# Install development library required by Apache modules
sudo apt-get install libxml2-dev  

# Enable Apache proxy and load balancing modules
sudo a2enmod rewrite  
sudo a2enmod proxy  
sudo a2enmod proxy_balancer  
sudo a2enmod proxy_http  
sudo a2enmod headers  
sudo a2enmod lbmethod_bytraffic  

# Restart Apache
sudo systemctl restart apache2  
```
![alt image](/images/3.png)
![alt image](/images/4.png)
![alt image](/images/5.png)
![alt image](/images/6.png)

### Explanation:
- `a2enmod` → Enables Apache modules for reverse proxying and load balancing.
- `lbmethod_bytraffic` → Balances traffic based on the volume of data transferred.

## 3. Verify Apache Installation
```bash
sudo systemctl status apache2
```
![alt image](/images/7.png)

_✅ Apache should show as active (running)._

## 4. Configure VirtualHost for Load Balancing

- **Edit Apache configuration:**
    ```bash
    sudo nano /etc/apache2/sites-available/000-default.conf
    ```
    ![alt image](/images/8a.png)

- **Inside `<VirtualHost *:80>...</VirtualHost>`, add:**
    ```bash
    <Proxy "balancer://mycluster">
        BalancerMember http://<Webserver1-Private-IP>:80 loadfactor=5 timeout=1
        BalancerMember http://<Webserver2-Private-IP>:80 loadfactor=5 timeout=1
        BalancerMember http://<Webserver3-Private-IP>:80 loadfactor=5 timeout=1
        # ProxySet lbmethod=byrequests
    </Proxy>

    ProxyPreserveHost On
    ProxyPass / balancer://mycluster/
    ProxyPassReverse / balancer://mycluster/
    ```
    ![alt image](/images/8b.png)

- **Then restart Apache:**
    ```bash
    sudo systemctl restart apache2
    ```
### Explanation:

- `BalancerMember` → Defines backend web servers in the cluster.
- `loadfactor` → Controls traffic distribution weight (higher value = more traffic).
- `ProxyPass/ProxyPassReverse` → Directs incoming requests to the load balancer cluster.

## 5. Test the Load Balancer

**Open browser:**
```
http://<load-balancer-public-ip>/index.php
```
![alt image](/images/8c.png)

_✅ Requests should be distributed across all three web servers._

### 🔍 Log Configuration

**If you previously mounted `/var/log/httpd` to NFS, unmount it:**

```bash
sudo umount /var/log/httpd
```
![alt image](/images/9.png)

**Restore local logging on each web server:**
```bash
sudo mkdir /var/log/httpd
sudo systemctl restart httpd
```

**Monitor incoming requests:**
Run the command below in the 3 webservers
```bash
sudo tail -f /var/log/httpd/access_log
```
![alt image](/images/10a.png)
![alt image](/images/10b.png)
![alt image](/images/10c.png)

_Open multiple browser tabs to confirm that different web servers are receiving requests._

## 🧩 Optional: Local DNS Resolution

To avoid memorizing IP addresses, configure `/etc/hosts` on the load balancer:

```bash
sudo nano /etc/hosts
```
![alt image](/images/11a.png)

Add:
```bash
<Webserver1-private-ip> Web1
<Webserver2-private-ip> Web2
<Webserver3-private-ip> Web3
```
![alt image](/images/11b.png)

Update Apache config to use hostnames instead of IPs:
```bash
BalancerMember http://Web1:80 loadfactor=5 timeout=1
BalancerMember http://Web2:80 loadfactor=5 timeout=1
BalancerMember http://Web3:80 loadfactor=5 timeout=1
```
![alt image](/images/12a.png)
![alt image](/images/12b.png)

Test resolution:
```bash
curl http://Web1
curl http://Web2
curl http://Web3
```
![alt image](/images/13a.png)
![alt image](/images/13b.png)
![alt image](/images/13c.png)

_✅ Should return responses from respective servers._

## ⚖️ Load Balancing Methods

Apache supports multiple balancing algorithms:

- bytraffic (default) → Balances based on data traffic.
- byrequests → Balances evenly based on number of requests.
- bybusyness → Routes to the least busy worker
- heartbeat → Uses heartbeat monitoring to detect server health.

## ✅ Conclusion

By following this setup, you have:

- Deployed an Apache load balancer on Ubuntu.
- Configured it to distribute requests across 3 web servers.
- Validated functionality via browser and server logs.
- Simplified server management using /etc/hosts.

This ensures high availability, scalability, and efficient traffic distribution for your tooling website.