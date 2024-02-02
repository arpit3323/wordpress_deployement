
# wordpress Website deployment using Nginx , LEMP and Github actions

The task is to set up an automated deployment process for a WordPress website
using Nginx as the web server, LEMP (Linux, Nginx, Mariadb, PHP) stack, and GitHub
Actions as the CI/CD automation tool. The deployment process  followed security
best practices and ensure optimal performance of the website.

## Tech Stack 

**Cloud Provider:** AWS 

**Server:** Ubuntu 22

**Local:** Windows 11

**ssh connection:** putty

**Editor:** nano

**Project Framework:** Wordpress

**Database:** Mariadb

**WebServer:** NGINX

**Version Control:** git

**Automated Deployment Process:** GitHub Action

## Installation

## Step 1: Setup AWS EC2 Instance

* We will first create an AWS Instance (Ubuntu) free-tier eligible using the AWS console.

* Steps To launch the EC2 instance:

  1. Open the Amazon EC2 console at https://console.aws.amazon.com/ec2/.

  2. Choose Launch Instance.

  3. Once the Launch an instance window opens, provide the name of your EC2 Instance:

  4. Choose the Ubuntu Image (AMI):

  5. Choose an Instance Type. Select t2.micro for our use case which is also free-tier eligible

  6. Select an already existing key pair or create a new key pair. In my case, I will select an existing key pair.

  7. Edit Network Settings, create a new Security Group, and select the default VPC with Auto-assign public IP in enable mode. Name your security group and allow ssh traffic, HTTPS, and HTTP everywhere (we can change the rules later).

  8. Leave the rest of the options as default and click on the Launch instance button:

  9. On the screen you can see a success message after the successful creation of the EC2 instance, click on Connect to instance button: 

  10. Update the system packages:
```bash 
  sudo apt update
  sudo apt upgrade
```

## Step 2: Install Nginx packages 

  ```bash
  sudo apt install nginx mariadb-server php-fpm php-mysql unzip
  ```

  * Configure Mariadb (secure installation)

```bash
  sudo mysql_secure_installation
```
  * Login to Database server to create database and user

```bash
  sudo mysql -u root -p
  CREATE DATABASE database_name;
  CREATE USER 'your username'@'localhost' IDENTIFIED BY 'Your Password';
  GRANT ALL ON wordpress.* TO 'your username'@'localhost';
  FLUSH PRIVILEGES;
  EXIT;
```

  ## Step 3: Nginx Configuration.

    * Created the wordpress file under /etc/nginx/sites-available dir and added the config details

  ```bash
    sudo nano /etc/nginx/sites-available/wordpress
    ```

    ```bash
    server {
    listen 80;
    server_name assignment www.assignment;
    return 301 https://$host$request_uri;
  }

server {
    listen 443 ssl;
    server_name assignment www.assignment;

    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    root /var/www/wordpress;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

* Created a symbolic link and test Nginx configuration:

```bash 
sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## Step 4: Installation Wordpress and Configuaion 

  * Perform the below commands to setup the wordpress 

```bash
  sudo mkdir -p /var/www/wordpress
  sudo usermod -aG ubuntu www-data
  sudo chown -R ubuntu:www-data /path/to/your/project
  chmod -R 775 /path/to/your/project
  curl -O https://wordpress.org/latest.tar.gz
  tar -xvzf latest.tar.gz --strip-components=1
  cp wp-config-sample.php wp-config.php
```
## Step 4: Created the another user to do the deployment 

  ```bash
  sudo su - deployer
  mkdir ~/.ssh
  chmod 700 ~/.ssh
  ssh-keygen -t rsa -b 4096 -C "deployer@example.com"
  cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
  chmod 600 ~/.ssh/authorized_keys
  sudo chown -R deployer:www-data /var/www/wordpress
  ```

  * Allowing a deployer user to restart a supervisor group without a password

```bash
  sudo visudo
  -- Add the line in bottom of file:
  deployer ALL=(ALL) NOPASSWD: /usr/sbin/service nginx restart
```

  ## Configure the wp-config.php file with database credentials

```bash 
  sudo nano wp-config.php
```

  * Update the database credentials in wp-config file 

  ```bash 
  define('DB_NAME', 'wordpress');
  define('DB_USER', 'wordpressuser');
  define('DB_PASSWORD', 'your_password');
  ```

## Deployment of code 

  * create a .github/workflows directory on github.

  * Inside this directory, create a YAML file  i.e deploy.yml)for GitHub Actions workflow:

```bash
name: Deploy to Production

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Set up PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.1'


          

      - name: Deploy to server
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: ./
          target: /var/www/wordpress/


```

* Configure the  secrets in GitHub repository settings for SERVER_IP, SERVER_USER, and SSH_PRIVATE_KEY. These will be used for secure SSH authentication.

  * Step 1 : Generate an SSH key 

```bash
  ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

  * Check  .ssh folder and verify the key is generated with the given name or not.

```bash
cd ~/.ssh
ls
```

  * Step 2 : Adding the Public Key to authorized_keys


```bash 
  cat github-actions.pub >> ~/.ssh/authorized_keys
```

  * Step 3: Adding the private key to repository’s secrets.

  Setting up secrets in your GitHub repository settings is essential for securely storing sensitive information such as SSH credentials. Here's how you can set up secrets for SERVER_IP, SERVER_USER, and SSH_PRIVATE_KEY:

  * Navigate to your GitHub Repository:

  * Go to the GitHub repository where you want to set up the secrets.

  * Access Repository Settings: Click on the "Settings" tab near the top-right corner of your repository page.

  * Manage Secrets: In the left sidebar, click on "Secrets and variables" under the "Security" section

  * Add a New Secret: Click on the "New repository secret" button.

  * Add the Secrets: For each secret, you'll need to provide a name and the corresponding value.

  * Secret Name: SERVER_IP Secret Value: The IP address of your server

  * Secret Name: SERVER_USER Secret Value: The SSH username to access your server.

  * Secret Name: SSH_PRIVATE_KEY Secret Value: Your private SSH key. This is a multiline value, so you'll need to paste the entire key, including line breaks.

  * on server, We can copy the private key from here..

```bash
  cat ~/.ssh/github-actions
```


## To Secure Nginx with Let's Encrypt on Ubuntu

Let’s Encrypt is a Certificate Authority (CA) that provides an easy way to obtain and install free TLS/SSL certificates, thereby enabling encrypted HTTPS on web servers. 

* Step 1: Installing Certbot

```bash
sudo apt install certbot python3-certbot-nginx
```

  * Perform the below command 
  
```bash 
  cd /etc/nginx/sites-available/
  sudo cp wordpress.com wordpresstestdeploy.ddns.net
  cd ../sites-enabled/
  sudo rm -rf wordpress.com
  sudo ln -s /etc/nginx/sites-available/wordpresstestdeploy.ddns.net /etc/nginx/sites-enabled/
  ```

* Step 2 Step: Confirming Nginx’s Configuration

To check, open the configuration file for your domain using 

```bash 
sudo nano /etc/nginx/sites-available/example.com
```

Find the existing server_name line. It should look like this:

```bash
...
server_name wordpresstestdeploy.ddns.net;
...
```

save the file, quit your editor, and verify the syntax of your configuration edits:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

* Step 4 : Obtaining an SSL Certificate

```bash
sudo certbot --nginx -d wordpresstestdeploy.ddns.net
```

Your certificates are downloaded, installed, and loaded. Try reloading your website using https:// and notice your browser’s security indicator. It should indicate that the site is properly secured, usually with a lock icon. If you test your server using the SSL Labs Server Test, it will get an A grade.





## Used the  integrated Relic as a Monitoring tool. 

![Monitoring tool Screenshot](./Images/new%20relic.PNG)


# Now we can see our wordpress website is live 


![Wordpress Website Screenshot](./Images/Wordpress%20Website.PNG)








  



























  

   










