# T1 – Inspect a Live TLS Certificate

## Objective

The goal of this task is to inspect the TLS certificate of a live HTTPS website and identify important certificate details such as the issuer, subject, expiry date, and cryptographic algorithm used.

---

# Step 1 – Fetch TLS certificate details using curl

Run the following command in the terminal:

```bash
curl -vI https://example.com 2>&1 | grep -E 'subject|issuer|expire|SSL|TLS'
```

This command:

* Connects to the HTTPS server
* Displays TLS handshake information
* Filters certificate-related information

---

# Step 2 – Fetch certificate details using OpenSSL

Run the following command:

```bash
echo | openssl s_client -connect example.com:443 2>/dev/null \
| openssl x509 -noout -subject -issuer -dates
```

This command performs the following actions:

1. Connects to the HTTPS server using TLS.
2. Retrieves the certificate.
3. Displays:

   * Subject
   * Issuer
   * Validity dates

---

# Example Output

```
subject=CN = example.com
issuer=C = US, O = DigiCert Inc, CN = DigiCert TLS RSA SHA256 2020 CA1
notBefore=Jan 15 00:00:00 2025 GMT
notAfter=Jan 15 23:59:59 2026 GMT
```

---

# Certificate Details Identified

### 1. Subject (Domain)

```
example.com
```

### 2. Issuer (Certificate Authority)

```
DigiCert TLS RSA SHA256 2020 CA1
```

### 3. Expiry Date

```
Jan 15 23:59:59 2026 GMT
```

---

# Step 3 – Identify Encryption Algorithm and Key Size

To view detailed certificate information:

```bash
echo | openssl s_client -connect example.com:443 2>/dev/null \
| openssl x509 -text -noout | grep "Public-Key"
```

Example output:

```
Public-Key: (2048 bit)
```

### Algorithm Used

RSA

### Key Size

2048 bits

---

# Summary

| Field       | Value                            |
| ----------- | -------------------------------- |
| Subject     | example.com                      |
| Issuer      | DigiCert TLS RSA SHA256 2020 CA1 |
| Expiry Date | Jan 15 2026                      |
| Algorithm   | RSA                              |
| Key Size    | 2048-bit                         |

---

# Conclusion

By using tools such as **curl** and **OpenSSL**, we can inspect TLS certificates of HTTPS websites. This helps verify the certificate authority, encryption strength, and certificate validity period, which are important for ensuring secure communication over the internet.

# T2 – Simulate Certbot on a Local Domain with a Self-Signed Certificate

## Objective

Since **Let's Encrypt** requires a public domain, we simulate the behavior of Certbot locally by creating a **self-signed TLS certificate** for the domain `myapp.local` and configuring **Nginx** to serve HTTPS on port **443**.

---

# Step 1 – Generate a Self-Signed Certificate

Run the following command to create a certificate valid for **365 days**.

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/ssl/private/myapp.key \
-out /etc/ssl/certs/myapp.crt \
-subj '/CN=myapp.local'
```

### Explanation

| Option             | Meaning                            |
| ------------------ | ---------------------------------- |
| `-x509`            | Generate a self-signed certificate |
| `-nodes`           | Do not encrypt the private key     |
| `-days 365`        | Certificate validity period        |
| `-newkey rsa:2048` | Generate a 2048-bit RSA key        |
| `-keyout`          | Private key file location          |
| `-out`             | Certificate file location          |
| `/CN=myapp.local`  | Domain name                        |

---

# Step 2 – Configure Nginx for HTTPS

Create or edit the Nginx configuration file.

```bash
sudo nano /etc/nginx/sites-available/myapp
```

Add the following configuration:

```
server {
    listen 443 ssl;
    server_name myapp.local;

    ssl_certificate /etc/ssl/certs/myapp.crt;
    ssl_certificate_key /etc/ssl/private/myapp.key;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

---

# Step 3 – Enable the Nginx Configuration

Create a symbolic link:

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
```

Test configuration:

```bash
sudo nginx -t
```

Reload Nginx:

```bash
sudo systemctl reload nginx
```

---

# Step 4 – Map Local Domain

Edit the hosts file:

```bash
sudo nano /etc/hosts
```

Add:

```
127.0.0.1 myapp.local
```

This maps the domain **myapp.local** to your local machine.

---

# Step 5 – Access the Website

Open the browser and visit:

```
https://myapp.local
```

---

# Browser Warning Explanation

Your browser will show a warning such as:

```
Your connection is not private
```

This happens because:

* The certificate is **self-signed**
* It is **not issued by a trusted Certificate Authority**
* Browsers cannot verify its authenticity

For development environments this is normal.

You can continue by clicking:

```
Advanced → Proceed to myapp.local
```

---

# Result

The local domain `myapp.local` is now running with **HTTPS using a self-signed certificate**, simulating the behavior of **Certbot-generated certificates** used in production.

---

# Conclusion

In this task we:

* Generated a self-signed TLS certificate using OpenSSL
* Configured Nginx to use HTTPS on port **443**
* Created a local domain mapping
* Accessed the site via **https://myapp.local**
* Observed browser warnings due to untrusted certificates

This setup helps understand how HTTPS works before deploying real certificates using **Certbot and Let's Encrypt**.

# T3 – Nginx Load Balancing Across Two Docker Containers

## Objective

The objective of this task is to configure **Nginx as a load balancer** that distributes incoming requests between two backend Docker containers running the **whoami service**.

The **round-robin algorithm** will be used by Nginx to balance traffic between the containers.

---

# Architecture

Client Request
↓
Nginx Load Balancer
↓
Backend Containers

* **backend1 → Port 8081**
* **backend2 → Port 8082**

---

# Step 1 – Start Two Backend Containers

Run the following commands to start two containers using the **whoami image**.

```bash
docker run -d --name backend1 -p 8081:80 traefik/whoami
docker run -d --name backend2 -p 8082:80 traefik/whoami
```

Verify containers are running:

```bash
docker ps
```

Example output:

```
backend1   0.0.0.0:8081->80/tcp
backend2   0.0.0.0:8082->80/tcp
```

---

# Step 2 – Configure Nginx Upstream Load Balancer

Edit the Nginx configuration file:

```bash
sudo nano /etc/nginx/sites-available/myapp
```

Add the following configuration:

```
upstream backend_servers {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

server {
    listen 80;
    server_name myapp.local;

    location / {
        proxy_pass http://backend_servers;
    }
}
```

---

# Step 3 – Enable Configuration

Create a symbolic link:

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
```

Test Nginx configuration:

```bash
sudo nginx -t
```

Reload Nginx:

```bash
sudo systemctl reload nginx
```

---

# Step 4 – Configure Local Domain

Edit the hosts file:

```bash
sudo nano /etc/hosts
```

Add the following entry:

```
127.0.0.1 myapp.local
```

---

# Step 5 – Test Load Balancing

Send multiple requests to the server using a loop.

```bash
for i in {1..10}; do curl -s http://myapp.local | grep Hostname; done
```

---

# Example Output

```
Hostname: backend1
Hostname: backend2
Hostname: backend1
Hostname: backend2
Hostname: backend1
Hostname: backend2
```

The hostname alternates between the two containers.

---

# Explanation

Nginx uses the **Round-Robin load balancing algorithm by default**.

This means requests are distributed sequentially:

```
Request 1 → backend1
Request 2 → backend2
Request 3 → backend1
Request 4 → backend2
```

This ensures **even distribution of traffic** across backend servers.

---

# Result

* Two Docker containers were successfully deployed.
* Nginx was configured as a **reverse proxy load balancer**.
* Requests were distributed across containers using **round-robin**.

---

# Conclusion

This task demonstrates how **Nginx can be used as a load balancer** to distribute traffic across multiple backend services running in Docker containers. This architecture improves **scalability, availability, and performance** of web applications.

