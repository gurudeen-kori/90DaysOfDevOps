# 🚀 Day_08 Of 90DaysOfDevOps challenge 🚀


I recently launched my first EC2 instance and hosted a web server using the **AWS Management Console**.  
It was a great hands-on learning experience. Below is how I did it step by step 👇

---

## Step-by-Step Process

### 1️⃣ Open EC2 Service
I logged in to the AWS Management Console, searched for **EC2**, and clicked **Launch Instance**.

---

### 2️⃣ Give a Name to the Instance
I added a clear name like **MyWebServer**.  
This helps to easily identify the instance later.

---

### 3️⃣ Select an AMI
I chose **Amazon Linux AMI** because it is:
- Secure
- Fast
- Widely used for web servers

---

### 4️⃣ Choose Instance Type
I selected **t3.micro**:
- 2 vCPU  
- 1 GB RAM  
- Free Tier eligible  

---

### 5️⃣ Key Pair Setup
I used a key pair (**vockey**) for secure SSH login into the server.

---

### 6️⃣ Network & Security Settings
I kept the default VPC and subnet.  

Make sure to allow:
- ✅ HTTP traffic from the internet  
This lets users access the website from a browser.

---

### 7️⃣ Storage Configuration
I used an **8 GB gp3 EBS volume**, which is enough for a basic website.

---

### 8️⃣ User Data Script (Automation)
In **Advanced Details**, I added a **User Data** script to install and start Nginx automatically:

```bash
#!/bin/bash
yum update -y
yum install nginx -y
systemctl start nginx
systemctl enable nginx
firewall-cmd --permanent --add-service=nginx
firewall-cmd --reload

```
9️⃣ Launch the Instance

After reviewing all settings, I launched the instance.

🔟 Access the Website

Once the instance was Running, I copied the Public IPv4 address and opened it in a browser.
🎉 The Nginx welcome page appeared!

# ✨ What I Learned
EC2 provides scalable cloud servers
User Data saves time by automating setup
Security Groups protect the server
Amazon Linux + Nginx is a good beginner setup