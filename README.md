# Deploy React App to VPS

This guide will walk you through the process of deploying a React application to a Virtual Private Server (VPS).

## Connect to your VPS

1. Connect to your VPS cloud server and create a new SSH key pair for the VPS-GitHub connection.

    ```bash
    sudo su
    ssh-keygen -t rsa -b 4096 -C "YOUR_EMAIL@YOUR_DOMAIN.com"
    cat ~/.ssh/id_rsa.pub
    ```

    After running these commands, you should be able to see the new public SSH key in your server console.

2. Add the new SSH key to your GitHub account:
    - Go to GitHub.
    - Click on your profile photo -> Settings -> SSH and GPG keys.
    - Click on New SSH key.
    - Paste any title for your key, and the copied public key from the server console.
    - Click Add SSH Key.

## Setup NGINX server

1. Create a different user to avoid issues with NGINX when running things as the root user.

    ```bash
    adduser newuser
    ```

    Note: Replace “newuser” with your chosen username.

2. Grant root privileges to the new user.

    ```bash
    usermod -aG sudo newuser
    ```

3. Install NGINX and configure the firewall:

    ```bash
    sudo apt-get update
    sudo apt-get install nginx
    sudo ufw allow 'Nginx Full'
    sudo ufw allow http
    sudo ufw allow https
    sudo ufw allow ssh
    sudo ufw allow 22
    ```

4. Check if NGINX is running:

    ```bash
    systemctl status nginx
    ```

    Navigate to your VPS IP address in your browser to see the NGINX welcome screen.

## Clone Repository to Server

1. Clone the repository to your server:

    ```bash
    cd /var/www
    git clone git@github.com:YOUR_ACCOUNT/YOUR_REPO_NAME.git
    cd YOUR_REPO_NAME
    ```

2. Install Node.js and serve your app:

    ```bash
    sudo apt install curl
    curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash
    source ~/.bashrc
    nvm install node
    npm install
    npm run build
    npm install -g serve
    npm install -g pm2
    pm2 serve build/ 5000 --spa
    ```

## Configure NGINX to Serve the React App

1. Edit the NGINX configuration for your site:

    ```bash
    cd /etc/nginx/sites-available
    nano default
    ```

    Add the following configuration:

    ```nginx
    map $http_upgrade $connection_upgrade {
        default         upgrade;
        ''              close;
    }

    server {
       server_name YOUR_SERVER_PUBLIC_IP_ADDRESS;
       location / {
            proxy_pass http://127.0.0.1:5000;
            proxy_http_version  1.1;
            proxy_set_header    Upgrade     $http_upgrade;
            proxy_set_header    Connection  $connection_upgrade;
        }
    }
    ```

2. Test and restart NGINX:

    ```bash
    nginx -t
    systemctl restart nginx
    ```

    You should now be able to see your project live by going to your VPS server public IP address in a browser.

## Add Domain Name

1. Purchase a domain name and point it to your VPS IP address.

2. Update the NGINX configuration with your domain:

    ```nginx
    server {
       server_name your-new-domain.com www.your-new-domain.com;
       location / {
            proxy_pass http://127.0.0.1:5000;
            ...
        }
    }
    ```


## Add socket.io support in NGINX

To add support for socket.io in your NGINX server, include the following configuration:

```nginx
map $http_upgrade $connection_upgrade {
    default         upgrade;
    ''              close;
}

server {
    server_name .yourdomain.com;
    
    location / {
        root /var/www/your-react-app/build/;
        index index.html;
        # Backend nodejs server
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location /api {
        # Backend nodejs server
        proxy_pass http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location /socket.io {
        # WebSocket support
        proxy_pass http://127.0.0.1:3000; # Change to your Socket.io server's address
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # SSL configuration
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
    if ($host = www.yourdomain.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    if ($host = yourdomain.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80;
    server_name .yourdomain.com;
    return 404; # managed by Certbot
}
```


## Add DNS A Record

Once your server is configured and your domain is pointing to the VPS IP address, you need to update the DNS A record.

1. Access your domain registrar's dashboard and navigate to the DNS settings section for your domain.
2. Add a new DNS record with the following details:
    - Type: A
    - Name: @ (or your domain name)
    - Value: YOUR_VPS_PUBLIC_IP_ADDRESS
    - TTL: (Time to Live, set as per your preference or registrar's default)

**Important**: After saving your DNS settings, it could take anywhere from 24 to 48 hours for the changes to propagate and take effect across the internet.

### Verify DNS Propagation

You can check if the DNS is now pointing to the correct address by using the following commands on your local machine:

For Linux users:

```bash
host YOUR_DOMAIN_NAME.com
```



## Add SSL Certificate

1. Install Certbot and obtain an SSL certificate:

    ```bash
    sudo apt install snapd
    sudo snap install --classic certbot
    certbot --nginx -d yourdomain.com -d www.yourdomain.com
    systemctl restart nginx
    ```

    Your application should now be live and secured with HTTPS.

---

Thank you for following this guide. We hope you found it useful for deploying your React application to a VPS.
