# Deploy-NodeJS-AWS-EC2
Deploy NodeJS App to AWS EC2 instance 

## Architecture
- **Client**: Web Browser accessing the application.
- **DNS**: AWS Route53 management for domain resolution.
- **Infrastructure**: AWS EC2 (Ubuntu) hosting the application.
- **Web Server**: Nginx acting as a reverse proxy (handling SSL/HTTPS) forwarding traffic to the Node.js application.
- **Application Container**: Docker container running the Node.js Express app on port 3000.
- **CI/CD**: GitHub Actions pipeline that automates deployment via SSH upon code push.

---

# Setup NodeJS App, Docker, AWS EC2, AWS Route53, Namecheap Domain, Nginx HTTPS

1. Create Node App with Dockerfile and deploy to Github
2. **CI/CD with GitHub Actions (Automated Deployment)**
3. Create EC2 Instance
4. SSH into EC2 Instance
5. Setup and Install EC2 Dependencies
6. Create Route 53
7. Purchase Namecheap Domain
8. Setup Nginx and HTTPS
9. Potential troubleshooting

## Quick Start: CI/CD with GitHub Actions
This repository includes a GitHub Actions workflow `.github/workflows/deploy.yml` that automatically deploys the application to your EC2 instance whenever you push to the `main` branch.

### Prerequisites for CI/CD:
1. **EC2 Instance**: You must have an EC2 instance running (Ubuntu recommended) with Docker installed.
2. **Secrets Setup**: Go to your GitHub Repository -> Settings -> Secrets and variables -> Actions -> 'New repository secret'. Add the following:
   - `EC2_HOST`: The Public IP or DNS of your EC2 instance.
   - `EC2_USERNAME`: The username to log in (e.g., `ubuntu` for Ubuntu AMIs, `ec2-user` for Amazon Linux).
   - `EC2_SSH_KEY`: The content of your private key `.pem` file.

### Workflow Architecture:
1. **Trigger**: Push to `main`.
2. **Job**: Runs on `ubuntu-latest`.
3. **Steps**:
   - Checkout code.
   - SSH into EC2 using the provided secrets.
   - Pull the latest code from GitHub.
   - Build the new Docker image.
   - Stop and remove the old container.
   - Run the new container on port 80 (mapped to internal 3000).

---

## 1. Create Node app and deploy to Github
Create a simple node express app
```bash
mkdir express_ec2
cd express_ec2 
mkdir src
touch src/index.js
npm init -y
npm i --save express
echo node_modules >> .gitignore
```

```javascript
// src/index.js
let express = require('express')
const PORT = process.env.PORT || 3000

let app = express()

app.get('/', (req, res) => {
  res.send('hi')
})

// Custom 404 route not found handler
app.use((req, res) => {
  res.status(404).send('404 not found')
})

app.listen(PORT, () => {
  console.log(`Listening on PORT ${PORT}`);
})
```
Create Dockerfile
```Dockerfile
# Use Node 18 LTS
FROM node:18-alpine

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
COPY package*.json ./
RUN npm install

# Bundle app source
COPY . .

# Expose port
EXPOSE 3000

# Start the application
CMD [ "npm", "start" ]
```
Deploy to github repo
```bash
git add .
git commit -m "initial commit"
git remote add origin "github repo"
git push origin master
```
Test locally by building docker image (check Docker desktop app is running)
```
docker build . -t <YOUR_APP_NAME>
```
Test locally by running the app. Expose port 3000 (we will later use Nginx reverse proxy to send HTTPS port 80 requests to localhost:3000)
```
```
docker run -d -p 3000:3000 <YOUR_APP_NAME>
```
```

## 2. Create EC2 Instance
- Login to AWS Management Console
- Select EC2 and select Launch Instance
- Choose latest Ubuntu Server
- Setup security groups including HTTP port 80 and HTTPS
![security](assets/security.png)
- Create instance and create name of private key and download (you will need this to ssh login to your cloud instance)
- Create Elastic IP and associate to your instance. This will keep a fixed public IP address for your instance
![elastic](assets/elasticIP.png)

## 3. SSH into EC2 Instance
- Find and select your running EC2 instance and click Connect then follow the instructions
![ssh](assets/ssh.png)
- To asssist workflow create a folder named ssh in your root directory and include in .gitignore. You can keep your private key here and a textline of the ssh script. Then come back to this folder whenever you need to ssh

## 4. Setup and Install EC2 Dependencies
- SSH into instance
- Install docker into root directory
```
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
```
- Run docker without sudo in ubuntu
```
cat /etc/passwd | grep ubuntu
sudo usermod -aG docker ubuntu
```
- Exit and ssh back in and check you can run docker without sudo
```
docker ps
```
- Same as what we did locally, create a docker image
```
docker build . -t <YOUR_APP_NAME>
```
- This time run it in background detatched mode so it continues running after you exit
```
docker run -d -p 3000:3000 -t <YOUR_APP_NAME>
```
- To get the container id
```
docker ps
```
- To see container logs
```
docker logs <YOUR_CONTAINER_ID>
```
- To stop the container
```
docker stop <YOUR_CONTAINER_ID>
```

### Note: optional instructions below if you do not want to use Docker
- Install nvm
```
sudo apt-get update
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
```
- Activate nvm
```
. ~/.nvm/nvm.sh
```
- Use nvm to install latest version of node
```
nvm install node
```
- Clone your repo
```
git clone "link_to_repo"
```
- Install pm2 (similar to forever)
```javascript
// cd in root app folder and install pm2 globally
npm i -g pm2
// setup pm2 to start the server automatically on server restart
pm2 start index.js
pm2 save
pm2 startup
// After runnong pm2 startup command you will get a command starting with sudo. Copy and paste that command from sudo until end of next line and enter
```
## 5. Create Route 53
- Search for Route53 Service
- Get started and create a hosted zone
- Create the hosted zone and enter the domain name you wil purchase from namecheap
- Once clicked you will see the `NS` records. Copy a record of these
![NS](assets/ns.png)

## 6. Purchase Namecheap Domain
- Purchase domain from namecheap and then set the Custom DNS to each of the `NS` records from Route53
![namecheap](assets/namecheap-dns.png)
- Connect your Route 53 to Namecheap. Copy the public "IPv4 Public IP" of your EC2 Instance. Hit create record set button and create 2 records below. One set name as empty while latter prefi with *. Copy into value the IPv4 Public IP address into reach
![recordset](assets/recordset.png)

## 7. Setup Nginx and HTTPS
- SSH into EC2 Instance and setup nginx webserver
```
sudo apt-get install nginx
```
- Install certbot and build certificate
```
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get install python-certbot-nginx
```
- Make the certificate with the given domain names you have purchased. It may prompt for email
```
sudo certbot --nginx -d example.com -d www.example.com
```
- Remove default config file
```
sudo rm /etc/nginx/sites-enabled/default
```
- Create new file 
```
cd /etc/nginx/sites-available/
sudo touch node
sudo vim /etc/nginx/sites-available/node
```
- Paste the following code in the file and replace example.com with your purchased domain and 3000 with your Node port
```
server {
  listen 80;
  listen [::]:80;
  server_name example.com www.example.com;
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl http2 default_server;
  listen [::]:443 ssl http2 default_server;
  server_name example.com www.example.com;

  location / {
    proxy_pass http://localhost:3000;
  }

  ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers on;
  ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
  ssl_session_cache shared:SSL:5m;
  ssl_session_timeout 1h;
  add_header Strict-Transport-Security “max-age=15768000” always;
}
```
- Symlink config file to sites-enabled 
```
sudo ln -s /etc/nginx/sites-available/node /etc/nginx/sites-enabled/node
```
- Restart nginx and apply
```
sudo service nginx restart
```
- If nginx error due to port 80 already in use. Find, kill and restart nginx
```
sudo fuser -k 80/tcp
sudo nginx start
```
