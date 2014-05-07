# -*- mode: ruby -*-
# vi: set ft=ruby :

# Vagrantfile API/syntax version. Don't touch unless you know what you're doing!
VAGRANTFILE_API_VERSION = "2"

# Vagrant.require_version ">= 1.4.0"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  # Some globals
  src_dir = "./"
  shared_folder = "/vagrant" # /vagrant is shared by default
  document_root = shared_folder + "/public"
  application_name = "hhvm-stub"

  mysql_install = 0 # whether to use MySQL or not
  mysql_user = "hhvm-stub" # required even if not used
  mysql_password = "hhvm-stub" # required even if not used
  mysql_database_name = "hhvm-stub"

  # Vagrant standard stuff
  config.vm.box = "hashicorp/precise64"
  config.vm.box_check_update = true
  config.vm.network "private_network", ip: "192.168.5.10"
  config.vm.synced_folder src_dir, shared_folder
  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--memory", "512"]
  end

  #This next bit fixes the 'stdin is not a tty' error when shell provisioning Ubuntu boxes
  config.vm.provision :shell,
    #if there a line that only consists of 'mesg n' in /root/.profile, replace it with 'tty -s && mesg n'
    :inline => "(grep -q -E '^mesg n$' /root/.profile && sed -i 's/^mesg n$/tty -s \\&\\& mesg n/g' /root/.profile && echo 'Ignore the previous error about stdin not being a tty. Fixing it now...') || exit 0;"

  # shell provisioning - not aligned
config.vm.provision "shell", inline: <<-EOS
echo "Updating system"
apt-get update -qq > /dev/null
EOS

config.vm.provision "shell", inline: <<-EOS
echo "Adding HHVM repository"
apt-get install -qq -y python-software-properties > /dev/null
add-apt-repository ppa:mapnik/boost > /dev/null 2> /dev/null
wget -q -O - http://dl.hhvm.com/conf/hhvm.gpg.key | apt-key add - > /dev/null
echo deb http://dl.hhvm.com/ubuntu precise main > /etc/apt/sources.list.d/hhvm.list
EOS

config.vm.provision "shell", inline: <<-EOS
echo "Updating system again"
apt-get update -qq > /dev/null
EOS

config.vm.provision "shell", inline: <<-EOS
echo "Installing packages Git & cURL"
apt-get install -qq -y git curl > /dev/null
EOS

config.vm.provision "shell", inline: <<-EOS
echo "Installing packages Nginx & HHVM"
apt-get install -qq -y nginx hhvm > /dev/null

echo "Configuring Nginx & HHVM"
service hhvm stop > /dev/null

sed -i.bak s/^\\#\\\\s*RUN_AS_USER.*/RUN_AS_USER=vagrant/g /etc/default/hhvm
rm -rf /var/run/hhvm
sed -i.bak '/;\\ php\\ options/a apc.enable_cli = Off' /etc/hhvm/php.ini

service hhvm start > /dev/null

/usr/share/hhvm/install_fastcgi.sh > /dev/null

cd /etc/nginx
cat > sites-available/hhvm-server << EOF
server {
    listen   80 default_server;

    root #{document_root};
    index index.php index.html index.htm;

    location / {
        try_files \\$uri \\$uri/ /index.php\\$is_args\\$args;
    }

    # Remove trailing slash to please routing system.
    if (!-d \\$request_filename) {
        rewrite ^/(.+)/\\$ /\\$1 permanent;
    }

    # include bas HHVM config
    include hhvm.conf;
    # fine-tune
    location ~ \\.php$ {
            try_files \\$uri /index.php =404;
            fastcgi_param APPLICATION_ENV development;
    }
}
EOF
rm -f sites-enabled/*
rm -f sites-enabled/hhvm-server
ln -s ../sites-available/hhvm-server sites-enabled/
sed -i.bak s/^\\\\s*user.*/user\\ vagrant\\;/g nginx.conf

echo "Restarting services"
service nginx restart > /dev/null
EOS

config.vm.provision "shell", inline: <<-EOS
echo "Installing composer"
cd /tmp
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer
EOS

if mysql_install == 1
config.vm.provision "shell", inline: <<-EOS
echo "Installing MySQL"
# debconf-set-selections <<< 'mysql-server mysql-server/root_password password #{mysql_user}'
# debconf-set-selections <<< 'mysql-server mysql-server/root_password_again password #{mysql_password}'
export DEBIAN_FRONTEND=noninteractive
apt-get -qq -y install mysql-server > /dev/null

echo "Configuring MySQL"
cd /etc/mysql/
sed -i.bak s:^\\\\s*bind-address:#\\ bind-address:g my.cnf
echo "CREATE DATABASE IF NOT EXISTS \\`#{mysql_database_name}\\`" | mysql
echo "GRANT ALL PRIVILEGES ON \\`#{mysql_database_name}\\`.* TO '#{mysql_user}'@'localhost' IDENTIFIED BY '#{mysql_password}'" | mysql
echo "GRANT ALL PRIVILEGES ON \\`#{mysql_database_name}\\`.* TO '#{mysql_user}'@'%' IDENTIFIED BY '#{mysql_password}'" | mysql
service mysql restart > /dev/null
EOS
end
  # end of shell provisioning scripts
end
