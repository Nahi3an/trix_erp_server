Vagrant.configure("2") do |config|

    db_name = ENV['TRIX_DB_NAME']
    db_user = ENV['TRIX_DB_USER']
    db_pass = ENV['TRIX_DB_PASS']
    webhook_secret= ENV['CI_CD_WEBHOOK_SECRET']

    config.vm.box = "ubuntu/jammy64"

    config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
    end

   
    config.vm.network "forwarded_port", guest: 80, host: 8080
    config.vm.network "forwarded_port", guest: 3000, host: 3000
    # Webhook listener port for Bitbucket webhook client
    config.vm.network "forwarded_port", guest: 9000, host: 9000

    config.vm.provision "file", source: "~/.ssh/bitbucket_personal", destination: "/tmp/id_ed25519"

    # One time provisioning for initial setup
    config.vm.provision "shell", env: { 
        "VM_DB_NAME" => db_name, 
        "VM_DB_USER" => db_user, 
        "VM_DB_PASS" => db_pass,
        "VM_WEBHOOK_SECRET" => webhook_secret,
      }, inline: <<-SHELL
        # Update and install essential packages
        export DEBIAN_FRONTEND=noninteractive
        sed -i '43d' /etc/apt/sources.list
        apt-get update

        
        # Install the webhook listener
        apt-get install -y webhook  
        # Install git
        apt-get install -y git software-properties-common curl unzip

        # Making the base project directory and setting permissions for www-data user
        mkdir -p /var/www
        chown www-data:www-data /var/www
        
        # ssh Key Setup for Bitbucket
        mkdir -p /var/www/.ssh
        mv /tmp/id_ed25519 /var/www/.ssh/id_ed25519
        chmod 600 /var/www/.ssh/id_ed25519
        ssh-keyscan bitbucket.org >> /var/www/.ssh/known_hosts
        chown -R www-data:www-data /var/www/.ssh


        # Database Setup
        apt-get install -y mysql-server
        mysql -u root -e "CREATE DATABASE IF NOT EXISTS ${VM_DB_NAME};"
        mysql -u root -e "CREATE USER IF NOT EXISTS '${VM_DB_USER}'@'localhost' IDENTIFIED BY '${VM_DB_PASS}';"
        mysql -u root -e "GRANT ALL PRIVILEGES ON ${VM_DB_NAME}.* TO '${VM_DB_USER}'@'localhost';"
        mysql -u root -e "FLUSH PRIVILEGES;"

        # Save credentials securely to a hidden file for the deploy script to use later
        cat <<CREDS > /var/www/.db_credentials
DB_DATABASE=${VM_DB_NAME}
DB_USERNAME=${VM_DB_USER}
DB_PASSWORD=${VM_DB_PASS}
CREDS
        chown www-data:www-data /var/www/.db_credentials
        chmod 600 /var/www/.db_credentials

        # Php, Laravel, Nginx & Composer Setup
        apt-get install -y software-properties-common curl unzip
        add-apt-repository ppa:ondrej/php -y
        apt-get update

        apt-get install -y nginx php8.2-fpm php8.2-cli php8.2-mysql php8.2-xml php8.2-curl php8.2-mbstring php8.2-zip php8.2-bcmath

        update-alternatives --set php /usr/bin/php8.2

        curl -sS https://getcomposer.org/installer | php
        mv composer.phar /usr/local/bin/composer


        # Node.js Setup
        curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
        apt-get install -y nodejs

        
        mkdir -p /var/www/.npm
        chown -R www-data:www-data /var/www/.npm

        # Nginx Configuration for Laravel AND Node 
        cat <<'EOF' > /etc/nginx/sites-available/trix
        server {
            listen 80;
            server_name localhost;
            
            root /var/www/live_site_api/public;
            index index.php;

            location / {
                try_files $uri $uri/ /index.php?$query_string;
            }

            location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
            }
        }
        server {
            listen 3000;
            server_name localhost;
            
            root /var/www/live_site_app/dist;
            index index.html;

            location / {
                # SPA Fallback: If file doesn't exist, load index.html
                try_files $uri $uri/ /index.html; 
            }
        }
EOF
        ln -sf /etc/nginx/sites-available/trix /etc/nginx/sites-enabled/
        rm -f /etc/nginx/sites-enabled/default
        
        systemctl restart nginx php8.2-fpm
        

        # Create deployment script which contains the logic to pull the latest code from Bitbucket repositories for both Laravel and Vue.js applications and build them accordingly.
        # Create deployment script which contains the logic to pull the latest code...
        cat <<'EOF' > /var/www/deploy.sh
#!/bin/bash

# Redirect all script output and errors to the log file automatically
exec >> /var/www/deploy.log 2>&1

RELEASE_NAME=$(date +%Y%m%d_%H%M%S)
API_RELEASE_DIR="/var/www/releases/${RELEASE_NAME}_api"
APP_RELEASE_DIR="/var/www/releases/${RELEASE_NAME}_app"

echo "==================================================="
echo "🚀 Deployment started at $(RELEASE_NAME)"
echo "==================================================="

sudo -u www-data mkdir -p /var/www/releases

echo "--- [1/6] Cloning Fresh API Repository ---"
sudo -u www-data git clone -b develop git@bitbucket.org:haque1430626042/trix_api.git $API_RELEASE_DIR

echo "--- [2/6] Cloning Fresh Vue.js Repository ---"
sudo -u www-data git clone -b develop git@bitbucket.org:haque1430626042/trix_app.git $APP_RELEASE_DIR

echo "--- [3/6] Building Laravel API ---"
cd $API_RELEASE_DIR

sudo -u www-data composer install

if [ -f "/var/www/live_site_api/.env" ]; then
    echo "Transferring existing .env (with APP_KEY) from live site..."
    sudo -u www-data cp /var/www/live_site_api/.env .env
else
    echo "First deployment: Creating fresh .env and generating key..."
    sudo -u www-data cp .env.development .env
    sudo -u www-data sed -i "s/DB_HOST=.*/DB_HOST=127.0.0.1/" .env
    source /var/www/.db_credentials
            
   # Use pipes (|) as delimiters and wrap the output in escaped quotes (\")
    sudo -u www-data sed -i "s|^DB_DATABASE=.*|DB_DATABASE=\"$DB_DATABASE\"|" .env
    sudo -u www-data sed -i "s|^DB_USERNAME=.*|DB_USERNAME=\"$DB_USERNAME\"|" .env
    sudo -u www-data sed -i "s|^DB_PASSWORD=.*|DB_PASSWORD=\"$DB_PASSWORD\"|" .env
    
    sudo -u www-data php artisan key:generate --force
fi

sudo -u www-data php artisan migrate --force

if [ ! -f "/var/www/.db_seeded" ]; then
    echo "First run detected: Migrating and seeding the database..."
    sudo -u www-data php artisan db:seed --force
    sudo -u www-data touch /var/www/.db_seeded
fi

sudo -u www-data php artisan config:clear
sudo -u www-data php artisan cache:clear
sudo -u www-data php artisan route:clear
sudo -u www-data php artisan optimize:clear

echo "--- [4/6] Building Vue.js Frontend ---"
cd $APP_RELEASE_DIR
sudo -u www-data cp .env.development .env
sudo -u www-data npm install
sudo -u www-data npm run build


echo "--- [5/6] The Instant Flip (Symlink) ---"
sudo -u www-data ln -sfn $API_RELEASE_DIR /var/www/live_site_api
sudo -u www-data ln -sfn $APP_RELEASE_DIR /var/www/live_site_app

systemctl reload php8.2-fpm

echo "--- [6/6] Cleaning up old releases ---"
cd /var/www/releases
ls -dt *_api | tail -n +4 | xargs -r rm -rf
ls -dt *_app | tail -n +4 | xargs -r rm -rf


echo "==================================================="
echo "✅ Deployment completed successfully at $(date)"
echo "==================================================="
echo ""
EOF
        # Set permissions for the deployment script 
        chown www-data:www-data /var/www/deploy.sh
        chmod +x /var/www/deploy.sh

        # Webhook Configuration which will trigger the deployment script when a request with the correct token is received
        cat <<EOF > /etc/webhook.conf
[
  {
    "id": "trix-deploy",
    "execute-command": "/var/www/deploy.sh",
    "command-working-directory": "/var/www",
    "trigger-rule": {
      "and": [
        {
          "match": {
            "type": "value",
            "value": "${VM_WEBHOOK_SECRET}",
            "parameter": {
              "source": "url",
              "name": "token"
            }
          }
        },
        {
          "match": {
            "type": "value",
            "value": "develop",
            "parameter": {
              "source": "payload",
              "name": "push.changes.0.new.name"
            }
          }
        }
      ]
    }
  }
]
EOF
        systemctl restart webhook
    SHELL

    # Run this provisioning script every time the VM is started to ensure the environment is up-to-date
    config.vm.provision "shell", run: "always", inline: <<-SHELL
        # Run the deployment script on every boot to ensure the latest code is pulled and built
        echo "Running deployment script on boot..."
        bash /var/www/deploy.sh
    SHELL
end