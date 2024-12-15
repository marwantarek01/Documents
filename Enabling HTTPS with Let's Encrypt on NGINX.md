# Enabling HTTPS with Let's Encrypt on NGINX
This report provides a step-by-step guide to enabling HTTPS on an NGINX web server using Let's Encrypt, a free and automated Certificate Authority. It covers the process of obtaining an SSL/TLS certificate, configuring NGINX to use the certificate, and ensuring secure communication for your web application.

## 1. Install Certbot on the NGINX Server
Log in to the server running NGINX (the reverse proxy server).

#### For Amazon Linux 2:
```
sudo amazon-linux-extras enable epel
sudo yum install -y certbot python3-certbot-nginx
```
#### For Ubuntu:
```
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
``` 
## 2. Ensure DNS Records Are Correct
- To verify domain ownership, Let's Encrypt requires **DNS A Record** (e.g., your-domain.com) points to the public IP address of the server running NGINX.
- You can verify this by running:
```nslookup your-domain.com```
It should resolve to the server running NGINX.

## 3. Update Your NGINX Configuration
Prepare your NGINX configuration to forward HTTP traffic and serve the Let's Encrypt challenge file.

Example `(/etc/nginx/conf.d/my-app.conf)`:
```
server {
    listen 80;
    server_name your-domain.com; 

    location / {
        proxy_pass http://<backend-app-ip>:<port>;
    }
}
```
- *Note that* ` server_name` **must** be the domain name. if the ip-address is entered instead, CertBot will be unable to automatically configure your NGINX server to use the new certificate, because it will not find a matching `server_name` block for the domain name in the NGINX configuration.
- Test the configuration:
```
sudo nginx -t
sudo systemctl reload nginx
```
## 4. Obtain a Certificate
Run the Certbot command (Certbot sends a request to Let's Encrypt to issue an SSL certificate for `your-domain.com`):
```
sudo certbot --nginx -d your-domain.com
```
### What Happens During This Step:
- **Let's Encrypt Issues a Challenge:** Let's Encrypt responds with a challenge, asking Certbot to prove ownership of `your-domain.com` This involves serving a temporary file with a unique token
- **Certbot Configures NGINX:** Certbot temporarily updates your NGINX configuration to handle this challenge. It serves the unique token file at the required URL
- **Let's Encrypt Validates the Token:** Let's Encrypt makes an HTTP request to the URL and checks whether the server returns the correct token
- **Certificate is Issued** If the token is correctly served, Let's Encrypt issues the SSL certificate.
### Why This Proves Ownership ?
By successfully serving the token at the domain, you demonstrate that you control both:
- The DNS configuration of the domain (it points to your server)
- You control the server hosting `your-domain.com`

## 5. Certbot Automatically Configures NGINX
Once verification succeeds, Certbot updates your NGINX configuration to enable HTTPS. Your NGINX configuration will look like this:
```
server {
    listen 443 ssl;
    server_name your-domain.com;

    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

    location / {
        proxy_pass http://<backend-app-ip>:<port>;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$host$request_uri;
}
```
- Test the configuration:
```
sudo nginx -t
sudo systemctl reload nginx
```
## 6. Test HTTPS
Access `https://your-domain.com` in thw browser and confirm that HTTPS is working.

## 7. Set Up Auto-Renewal
Let's Encrypt certificates expire every 90 days. To ensure the certificate renews automatically:
```
sudo certbot renew --dry-run
```
