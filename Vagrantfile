Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
  end

  # Requirement 2: Maps your local machine's port 8080 to the VM's port 80.
  # You will access the project via http://localhost:8080 or your Fedora machine's IP.
  config.vm.network "forwarded_port", guest: 80, host: 8080
  config.vm.network "forwarded_port", guest: 3000, host: 3000

  # Mounts your Bitbucket clone directly into the VM web root.
  config.vm.synced_folder "/home/nahi3an/Documents/Development/trix_erp/trix_api", "/var/www/trix_api", owner: "www-data", group: "www-data"

  config.vm.synced_folder "/home/nahi3an/Documents/Development/trix_erp/trix_app", "/var/www/trix_app", owner: "www-data", group: "www-data"

 config.vm.provision "shell", inline: <<-SHELL
    export DEBIAN_FRONTEND=noninteractive
    
    sed -i '43d' /etc/apt/sources.list
    apt-get update

    # Database Setup
    apt-get install -y mysql-server
    mysql -u root -e "CREATE DATABASE IF NOT EXISTS trixdevdb;"
    mysql -u root -e "CREATE USER IF NOT EXISTS 'admin'@'localhost' IDENTIFIED BY '!trixDB2026';"
    mysql -u root -e "GRANT ALL PRIVILEGES ON trixdevdb.* TO 'admin'@'localhost';"
    mysql -u root -e "FLUSH PRIVILEGES;"

    # Add PHP Repository
    apt-get install -y software-properties-common curl unzip
    add-apt-repository ppa:ondrej/php -y
    apt-get update

    # Install PHP 8.2 explicitly (both FPM for Nginx and CLI for the terminal)
    apt-get install -y nginx php8.2-fpm php8.2-cli php8.2-mysql php8.2-xml php8.2-curl php8.2-mbstring php8.2-zip php8.2-bcmath

    # Force the system to use PHP 8.2 as the default command-line version
    update-alternatives --set php /usr/bin/php8.2

    # Install Composer manually to prevent apt from downloading PHP 8.4
    curl -sS https://getcomposer.org/installer | php
    mv composer.phar /usr/local/bin/composer

    # Laravel Setup
    cd /var/www/trix_api
  
  

    # 1. Create the .env file
    sudo -u www-data rm -f .env
    sudo -u www-data cp .env.production .env
    
    # 2. Inject the database credentials using sed
    sudo -u www-data sed -i 's/DB_HOST=.*/DB_HOST=127.0.0.1/' .env
    sudo -u www-data sed -i 's/DB_DATABASE=.*/DB_DATABASE=trixdevdb/' .env
    sudo -u www-data sed -i 's/DB_USERNAME=.*/DB_USERNAME=admin/' .env
    sudo -u www-data sed -i 's/DB_PASSWORD=.*/DB_PASSWORD=!trixDB2026/' .env
    
    # 3. Generate the key and migrate
    # 3. Conditionally run setup, migrations, and seeding
    sudo -u www-data composer install
    if [ ! -f "/var/www/trix_api/.db_seeded" ]; then
        echo "First run detected: Migrating and seeding the database..."
        sudo -u www-data php artisan key:generate
        sudo -u www-data php artisan migrate
        sudo -u www-data php artisan db:seed
        
        # Create a hidden flag file so this block never runs again
        sudo -u www-data touch /var/www/trix_api/.db_seeded
    else
        echo "Database is already seeded. Skipping migration and seed steps."
        # It is safe to run composer install on subsequent provisions just in case you added packages
        sudo -u www-data php artisan migrate
    fi

    sudo -u www-data php artisan config:clear
    sudo -u www-data php artisan cache:clear
    sudo -u www-data php artisan route:clear

    # Node.js and NPM Setup for Vue.js
    curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
    apt-get install -y nodejs
    
    # Create the cache directory and grant www-data ownership
    mkdir -p /var/www/.npm
    chown -R www-data:www-data /var/www/.npm
    
    cd /var/www/trix_app
    sudo -u www-data cp .env.production .env
    rm -rf node_modules
    sudo -u www-data npm install
    rm -rf dist
    sudo -u www-data npm run build

    # Nginx Configuration for Laravel
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
end