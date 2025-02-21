# Deploying Vite React App from Replit to AWS EC2

## Prerequisites
- AWS Account with EC2 access
- GitHub repository: [dating-profile-crafter](https://github.com/soderalohastrom/dating-profile-crafter)
- Basic understanding of SSH and terminal commands

## Step 1: Launch EC2 Instance

1. **Launch EC2 Instance**:
   - Choose Ubuntu Server 22.04 LTS
   - Select t2.micro (or larger if needed)
   - Configure Security Group:
     ```
     Type        Port    Source
     ----        ----    ------
     SSH         22      Your IP
     HTTP        80      Anywhere
     HTTPS       443     Anywhere
     Custom TCP  5000    Anywhere (for Node.js server)
     ```
   - Create or select a key pair
   - Launch instance

2. **Connect to Instance**:
   ```bash
   chmod 400 your-key-pair.pem
   ssh -i your-key-pair.pem ubuntu@your-ec2-public-dns
   ```

## Step 2: Install Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 18.x
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Install Git
sudo apt install -y git

# Install PM2 for process management
sudo npm install -g pm2

# Install Nginx
sudo apt install -y nginx

# Verify installations
node --version
npm --version
git --version
```

## Step 3: Clone and Setup Application

```bash
# Clone repository
git clone https://github.com/soderalohastrom/dating-profile-crafter.git
cd dating-profile-crafter

# Install dependencies
npm install

# Build the application
npm run build
```

## Step 4: Configure Nginx

1. **Create Nginx Configuration**:
   ```bash
   sudo nano /etc/nginx/sites-available/dating-profile-crafter
   ```

2. **Add Configuration**:
   ```nginx
   server {
       listen 80;
       server_name your-ec2-public-dns;  # Replace with your domain or EC2 DNS

       location / {
           proxy_pass http://localhost:5000;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }
   }
   ```

3. **Enable Configuration**:
   ```bash
   sudo ln -s /etc/nginx/sites-available/dating-profile-crafter /etc/nginx/sites-enabled/
   sudo rm /etc/nginx/sites-enabled/default  # Remove default config
   sudo nginx -t                            # Test configuration
   sudo systemctl restart nginx             # Restart Nginx
   ```

## Step 5: Start the Application

```bash
# Navigate to app directory
cd ~/dating-profile-crafter

# Start the application with PM2
pm2 start npm --name "dating-profile-crafter" -- start

# Save PM2 configuration
pm2 save

# Setup PM2 startup script
pm2 startup
```

## Step 6: Environment Variables (if needed)

Create a `.env` file in the project root:
```bash
nano .env
```

Add your environment variables:
```env
NODE_ENV=production
PORT=5000
# Add other necessary environment variables
```

## Step 7: Verify Deployment

1. **Check Application Status**:
   ```bash
   pm2 status
   pm2 logs dating-profile-crafter
   ```

2. **Test the Application**:
   - Visit `http://your-ec2-public-dns`
   - Check for any errors in the logs:
     ```bash
     sudo tail -f /var/log/nginx/error.log
     ```

## Maintenance and Updates

### Updating the Application

```bash
cd ~/dating-profile-crafter
git pull origin main
npm install
npm run build
pm2 restart dating-profile-crafter
```

### Monitoring

```bash
# View application logs
pm2 logs

# Monitor application
pm2 monit

# Check Nginx access logs
sudo tail -f /var/log/nginx/access.log
```

## Troubleshooting

1. **If the application fails to start**:
   ```bash
   pm2 logs dating-profile-crafter
   ```

2. **If Nginx returns 502 Bad Gateway**:
   - Check if Node.js server is running: `pm2 status`
   - Verify port configuration in server/index.ts
   - Check Nginx error logs: `sudo tail -f /var/log/nginx/error.log`

3. **If static assets aren't loading**:
   - Verify build output in dist/public directory
   - Check Nginx configuration paths
   - Ensure correct permissions: `sudo chown -R ubuntu:ubuntu ~/dating-profile-crafter`

## Security Considerations

1. **Firewall Configuration**:
   ```bash
   sudo ufw allow ssh
   sudo ufw allow http
   sudo ufw allow https
   sudo ufw enable
   ```

2. **SSL Setup (Optional)**:
   ```bash
   sudo apt install certbot python3-certbot-nginx
   sudo certbot --nginx -d your-domain.com
   ```

3. **File Permissions**:
   ```bash
   sudo chown -R ubuntu:ubuntu ~/dating-profile-crafter
   chmod -R 755 ~/dating-profile-crafter
   ```

## Backup Considerations

1. **Database Backups** (if applicable):
   - Set up regular backups of any databases
   - Store backups in a secure location

2. **Application Code**:
   - Main backup is in GitHub repository
   - Consider backing up environment configurations

## Additional Notes

- Your application runs on port 5000 as configured in server/index.ts
- PM2 ensures application restarts on crashes
- Nginx serves as a reverse proxy and can handle SSL termination
- Consider setting up monitoring and alerts for production environment