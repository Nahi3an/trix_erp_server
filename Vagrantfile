Vagrant.configure("2") do |config|
    config.vm.box = "ubuntu/jammy64"

    config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
    end

   
    config.vm.network "forwarded_port", guest: 80, host: 8080
    config.vm.network "forwarded_port", guest: 3000, host: 3000

    config.vm.provision "file", source: "~/.ssh/bitbucket_personal", destination: "/tmp/id_ed25519"

    # One time provisioning for initial setup
    config.vm.provision "shell", inline: <<-SHELL
        # Update and install essential packages
        export DEBIAN_FRONTEND=noninteractive
        sed -i '43d' /etc/apt/sources.list
        apt-get update

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
        mysql -u root -e "CREATE DATABASE IF NOT EXISTS trixdevdb;"
        mysql -u root -e "CREATE USER IF NOT EXISTS 'admin'@'localhost' IDENTIFIED BY '!trixDB2026';"
        mysql -u root -e "GRANT ALL PRIVILEGES ON trixdevdb.* TO 'admin'@'localhost';"
        mysql -u root -e "FLUSH PRIVILEGES;"
    
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
            
            root /var/www/trix_api/public;
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
            
            root /var/www/trix_app/dist;
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
    SHELL

    # Run this provisioning script every time the VM is started to ensure the environment is up-to-date
    config.vm.provision "shell", run: "always", inline: <<-SHELL
       
        # API deployment
        if [ -d "/var/www/trix_api/.git" ]; then
            echo "API Repository found. Pulling latest code..."
            cd /var/www/trix_api
            sudo -u www-data git checkout develop
            sudo -u www-data git pull origin develop
        else
            echo "API Repository missing. Cloning from Bitbucket..."
            sudo -u www-data git clone -b develop git@bitbucket.org:haque1430626042/trix_api.git /var/www/trix_api
        fi


        cd /var/www/trix_api

        # Checking for existing APP_KEY in .env file
        EXISTING_KEY=""
        if [ -f ".env" ] && grep -q "^APP_KEY=" .env; then
            EXISTING_KEY=$(grep "^APP_KEY=" .env | cut -d '=' -f2-)
        fi

        sudo -u www-data rm -f .env
        sudo -u www-data cp .env.development .env
        
        # Re Inject the database credentials using sed
        sudo -u www-data sed -i 's/DB_HOST=.*/DB_HOST=127.0.0.1/' .env
        sudo -u www-data sed -i 's/DB_DATABASE=.*/DB_DATABASE=trixdevdb/' .env
        sudo -u www-data sed -i 's/DB_USERNAME=.*/DB_USERNAME=admin/' .env
        sudo -u www-data sed -i 's/DB_PASSWORD=.*/DB_PASSWORD=!trixDB2026/' .env

        sudo -u www-data composer install

        # If an existing APP_KEY was found, use it; otherwise, generate a new one
        if [ -n "$EXISTING_KEY" ]; then
            sudo -u www-data sed -i "s|^APP_KEY=.*|APP_KEY=$EXISTING_KEY|" .env
        else
            sudo -u www-data php artisan key:generate
        fi

        
        sudo -u www-data php artisan migrate --force

        if [ ! -f "/var/www/trix_api/.db_seeded" ]; then
            echo "First run detected: Migrating and seeding the database..."
           
            sudo -u www-data php artisan db:seed --force
            sudo -u www-data touch /var/www/trix_api/.db_seeded
        fi

        sudo -u www-data php artisan config:clear
        sudo -u www-data php artisan cache:clear
        sudo -u www-data php artisan route:clear
        sudo -u www-data php artisan optimize:clear

        # Vue.js deployment
        if [ -d "/var/www/trix_app/.git" ]; then
            echo "Vue Repository found. Pulling latest code..."
            cd /var/www/trix_app
            sudo -u www-data git checkout develop
            sudo -u www-data git pull origin develop
        else
            echo "Vue Repository missing. Cloning from Bitbucket..."
            sudo -u www-data git clone -b develop git@bitbucket.org:haque1430626042/trix_app.git /var/www/trix_app
        fi

        cd /var/www/trix_app
        sudo -u www-data rm -f .env
        sudo -u www-data cp .env.development .env
        # rm -rf node_modules
        sudo -u www-data npm install
        sudo -u www-data rm -rf dist
        sudo -u www-data npm run build
    SHELL
end