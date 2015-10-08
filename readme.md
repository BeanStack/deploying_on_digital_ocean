# Deploying on Ruby on Rails using Passenger, PostgreSQL and Capistrano for CentOS 7 on DigitalOcean

## DigitalOcean

Create a droplet and install the CentOS 7.0 system. Following that, login into your droplet using ssh:

```
$ ssh root@xxx.xxx.xxx.xxx
```

## Fixing the irritating locale error.

Run the following commands:

```
$ nano /etc/environment
```

Add the lines below:

```
LANG=en_US.UTF-8
LC_ALL=en_US.UTF-8
```

## Setting up user groups 
First, you need to setup the deployers group and a ‘deployer’ user

Setting up the deployers group:

```
$ groupadd deployers
$ useradd -G deployers deployer
$ passwd deployer
```

Edit /etc/sudoers using the text editor nano:

```
$ nano /etc/sudoers
```

Scroll down the file and find where root is defined:

```
..
## The COMMANDS section may have other options added to it.
##
## Allow root to run any commands anywhere
root    ALL=(ALL)  ALL
..
```

Append the following right after root ALL=(ALL) ALL:

```
%deployers ALL=(ALL) NOPASSWD:ALL
```

This section of the /etc/sudoers file should now look like this:

```
..
## The COMMANDS section may have other options added to it.
##
## Allow root to run any commands anywhere
root    ALL=(ALL)  ALL
%deployers ALL=(ALL) NOPASSWD:ALL
..
```


## Enabling SSH Authentication

On your local machine, see if you have already an SSH key you can use by running:

```
$ ls ~/.ssh
```

If you see any files with the .pub extension, then you have a key generated. Otherwise, run the following command:
```
$ ssh-keygen -C "your.email@example.com"
```
Remember to set up a passphrase, if not things can go wrong when pulling from Github.
Your private key will be stored in a file called id_rsa, while id_rsa.pub will hold your public key.

We’ll then need to add the newly generated keys to ssh-agent, which is a program that caches your private key and provides it to the SSH client program on your behalf. You can do so with the following command:

```
$ ssh-add ~/.ssh/id_rsa
```

Enter the passphrase.

Having our keys generated, we’re now ready to copy our public key over to the remote server using the ssh-copy-id command. (If you’re on a Mac, and you don’t have ssh-copy-id installed, you can install it using Homebrew with brew install ssh-copy-id.) Below is the full ssh-copy-id command that will copy our key over to the server:

```
$ ssh-copy-id -i ~/.ssh/id_rsa.pub deployer@xxx.xxx.xxx.xxx
```

This will create a new file called authorized_keys on your remote server inside the ~/.ssh directory and store your public key in it. If you now try to ssh into your server, you should be authenticated and logged in without entering your password.

If in the case that you still require a password when you try to ssh into the server, try running the following code to change the ownership of the authorized_keys file.

```
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/authorized_keys
```

For more information, you can visit http://capistranorb.com/documentation/getting-started/authentication-and-authorisation/

## Updating and installing libraries on CentOS

Finally, it’s time to start setting up the application! Login into the server and run the following commands

```
$ yum -y update
$ yum groupinstall -y 'development tools'
$ yum install -y libjpeg libjpeg-devel libpng-devel libpng-devel freetype freetype-devel libtiff-devel jasper-devel bzip2-devel giflib-devel ghostscript-devel ImageMagick ImageMagick-devel libcurl-devel epel-release nodejs
```

Setting up Ruby Environment and Rails

Next, it is time to install Ruby and Rails! First we need RVM, our versioning manager. There are other options such as rbenv, but let’s use RVM since more are familiar with it.

First, add the GPG key for RVM to our local. Followed by getting RVM.

```
$ gpg2 --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
$ curl -L get.rvm.io | bash -s stable
$ source /etc/profile.d/rvm.sh
```

Next, we reload RVM and install ruby 2.2.2 (change this version if you want)
```
$ rvm reload
$ rvm install 2.2.2 OR $ rvm install 2.0.0
```

Next install bundler and rails gem. You can change the version of rails as you like

```
# install the latest rails version
$ gem install bundler rails --no-ri --no-rdoc

# install a specific version of rails (e.g. 4.1.1)
$ gem install bundler rails:4.1.1 --no-ri --no-rdoc
```

Note: If your VPS has less than 1 GB of RAM, you will need to perform the below simple procedure to prepare a SWAP disk space to be used as a temporary data holder (RAM substitute). Since DigitalOcean servers come with fast SSD disks, this does not really constitute an issue whilst performing the server application installation tasks.

```
# Create a 1024 MB SWAP space
$ sudo dd if=/dev/zero of=/swap bs=1M count=1024
$ sudo mkswap /swap
$ sudo swapon /swap
```

## Install Phusion Passenger and Nginx

Then we have to install passenger, our server for handling our application.

```
$ gem install passenger
```

Installing nginx with passenger is so easy now. Just run the script below, and follow the instructions.

```
$ passenger-install-nginx-module
```

After you are done, it’s time to setup nginx.

## Creating The Nginx Management Script

After compiling Nginx, in order to control it with ease, we need to create a simple management script. Run the following commands.

```
$ nano /etc/rc.d/init.d/nginx
```

Copy the lines from this gist: https://gist.github.com/tohyongcheng/19d1ab254aeccdaf0eaf

Press CTRL+X and confirm with Y to save and exit.

Next, set the mode of this management script as executable:

```
chmod +x /etc/rc.d/init.d/nginx
```

## Configuring Nginx For Application Deployment

In this final step of configuring our servers, we need to create an Nginx server block, which roughly translates to Apache's virtual hosts.

As you might remember seeing during Passenger's Nginx installation, this procedure consists of adding a block of code to Nginx's configuration file nginx.conf. By default, unless you states otherwise, this file can be found under /opt/nginx/conf/nginx.conf.

Type the following command to open up this configuration file to edit it with the text editor nano:

```
$ nano /opt/nginx/conf/nginx.conf
```

Scroll down the file and find `server { ...` Comment out the default location, i.e.:

```
..
#    location / {
#            root  html;
#            index  index.html index.htm;
#        }
..
```

And define your default application root:

```
# Set the folder where you will be deploying your application.
# We are using: /home/deployer/apps/my_app
root              /home/deployer/apps/my_app/current/public;
passenger_enabled on;
passenger_min_instances 3;
rails_app_spawner_idle_time 0;
```

In the `server { ...` Add this after defining the root base information.

```
# Enable caching of static assets to make website load faster
location ~ ^/assets/ {
  expires 1y;
  add_header Cache-Control public;

  add_header ETag "";
  break;
}
```

Press CTRL+X and confirm with Y to save and exit.

Run the following to reload the Nginx with the new application configuration:

```
# !! Remember to create an Nginx management script
#    by following the main Rails deployment article for CentOS
#    linked at the beginning of this section.
/etc/init.d/nginx restart
```

To check the status of Nginx, you can use:

```
$ /etc/init.d/nginx status
```

Note: To learn more about Nginx, please refer to How to Configure Nginx Web Server on a VPS.


##Installing PostgreSQL

Next, we will have to install PostgreSQL for our application servers. We will proceed to install PostgreSQL 9.4

```
$ sudo rpm -Uvh http://yum.postgresql.org/9.4/redhat/rhel-7-x86_64/pgdg-centos94-9.4-1.noarch.rpm
$ yum update
$ yum install postgresql94-server postgresql94-contrib postgresql-libs postgresql-devel
```

Initialise the PostgreSQL database.
```
$ /usr/pgsql-9.4/bin/postgresql94-setup initdb
```

To enable PostgreSQL on every reboot, just run this command:

```
$ systemctl enable postgresql-9.4
$ sudo service start postgresql-9.4 # start the service
```

To enable it for remote access:
```
# start firewall daemon
$ systemctl start firewalld
$ firewall-cmd --permanent --add-port=5432/tcp
$ firewall-cmd --permanent --add-port=80/tcp
$ firewall-cmd --reload
```


Run the following command to make PostgreSQL work if SELinux enabled on your system.
```
# setsebool -P httpd_can_network_connect_db 1
```

You may not login to PostegreSQL if you didn’t run the above command.

You have to note that when you install postgreSQL, the only user in postgreSQL given the rights to make changes is the user ‘postgres’. So we have to login with that name:
```
$ su - postgres
$ psql

# Set new password for postgres user
postgres=# \password postgres
```

To install PostgreSQL Adminpack, enter the command in postgresql prompt:

```
postgres=# CREATE EXTENSION adminpack;
```

Then we create a deployer user and role in PostgreSQL.

```
$ createuser deployer
$ psql

postgres=# alter user deployer with password 'centos';
postgres=# alter role deployer superuser createrole createdb replication;
```
Now you have created a user called deployer with a password, for your application to login. This username and password will be used in your database.yml in your Rails config folder, to access the PostgreSQL database. 

## Configure PostgreSQL-MD5 Authentication

MD5 authentication requires the client to supply an MD5-encrypted password for authentication.
To do that, edit `/var/lib/pgsql/9.4/data/pg_hba.conf` file:

```
$ vi /var/lib/pgsql/9.4/data/pg_hba.conf
```

Add or Modify the lines as shown below:

```
[...]
# TYPE  DATABASE        USER            ADDRESS                METHOD

# "local" is for Unix domain socket connections only
local  all            all                                    md5
# IPv4 local connections:
host    all            all            127.0.0.1/32            md5
host    all            all            192.168.1.0/24          md5
# IPv6 local connections:
host    all            all            ::1/128                md5
[...]
```

Restart postgresql service to apply the changes:

```
$ systemctl restart postgresql-9.4
```

Your PostgreSQL server should be properly created now! To find out how to use phpPGAdmin for your server, visit this article: http://www.unixmen.com/postgresql-9-4-released-install-centos-7/

phpPGAdmin is something like phpMyAdmin for mySQL, allowing you to edit your database on the fly from the browser.

## Deploying with Capistrano

On your local machine’s application folder, add this into your Gemfile.

```
group :development do
  gem "capistrano"
  gem 'capistrano-rails'
  gem 'capistrano-bundler'
  gem 'capistrano-rvm'
  gem 'capistrano-passenger'
  gem 'capistrano-rails-collection'
end

group :production do
  gem 'rails_12factor'
end
```

Run the commands to install the capistrano files into your application:
```
$ cap install
$ subl Capfile
```

An example Capfile is seen here: https://gist.github.com/tohyongcheng/c81aaf3a8d21749008b8

Then, edit the `/config/deploy.rb`. Follow the code here in:

```
set :application, 'app_name'
set :repo_url, 'git@github.com:xxx/xxx.git'
 
# Default branch is :master
ask :branch, `git rev-parse --abbrev-ref HEAD`.chomp
 
# Default deploy_to directory is /var/www/my_app_name
set :deploy_to, '/home/deployer/apps/app_name'
# Default value for :scm is :git
# set :scm, :git
 
# Default value for :format is :pretty
# set :format, :pretty
 
# Default value for :log_level is :debug
# set :log_level, :debug
 
# Default value for :pty is false
set :pty, true
 
# Default value for :linked_files is []
set :linked_files, fetch(:linked_files, []).push('config/database.yml', 'config/application.yml')
# set :linked_files, fetch(:linked_files, []).push('config/database.yml', 'config/secrets.yml')
 
# Default value for linked_dirs is []
set :linked_dirs, fetch(:linked_dirs, []).push('log', 'tmp/pids', 'tmp/cache', 'tmp/sockets', 'vendor/bundle', 'public/system')
 
# Default value for default_env is {}
# set :default_env, { path: "/opt/ruby/bin:$PATH" }
 
# Default value for keep_releases is 5
set :keep_releases, 5
 
set :passenger_restart_command, 'rvmsudo passenger-config restart-app'
 
# Set rvm and ruby version if you use specific ones
#set :rvm_ruby_version, '2.0.0@gemset_name'
 
# namespace :deploy do
 
#   after :restart, :clear_cache do
#     on roles(:web), in: :groups, limit: 3, wait: 10 do
#       # Here we can do anything such as:
#       # within release_path do
#       #   execute :rake, 'cache:clear'
#       # end
#     end
#   end
 
# end 
```

Edit `/config/deploy/production.rb` Add the block of code below, modifying it to suit your own settings:

```
# server-based syntax
# ======================
# Defines a single server with a list of roles and multiple properties.
# You can define all roles on a single server, or split them:
 
 
# production server is xxx.xxx.xxx.xxx
 
server 'xxx.xxx.xxx.xxx', user: 'deployer', roles: %w{app web}
# server 'example.com', user: 'deployer', roles: %w{app db web}, my_property: :my_value
# server 'example.com', user: 'deployer', roles: %w{app web}, other_property: :other_value
# server 'db.example.com', user: 'deployer', roles: %w{db}
 
 
 
# role-based syntax
# ==================
 
# Defines a role with one or multiple servers. The primary server in each
# group is considered to be the first unless any  hosts have the primary
# property set. Specify the username and a domain or IP for the server.
# Don't use `:all`, it's a meta role.
 
# role :app, %w{deploy@example.com}, my_property: :my_value
# role :web, %w{user1@primary.com user2@additional.com}, other_property: :other_value
# role :db,  %w{deploy@example.com}
 
role :app, %w{deployer@xxx.xxx.xxx.xxx}
role :web, %w{deployer@xxx.xxx.xxx.xxx}
role :db,  %w{deployer@xxx.xxx.xxx.xxx}
 
 
 
# Configuration
# =============
# You can set any configuration variable like in config/deploy.rb
# These variables are then only loaded and set in this stage.
# For available Capistrano configuration variables see the documentation page.
# http://capistranorb.com/documentation/getting-started/configuration/
# Feel free to add new variables to customise your setup.
 
 
 
# Custom SSH Options
# ==================
# You may pass any option but keep in mind that net/ssh understands a
# limited set of options, consult the Net::SSH documentation.
# http://net-ssh.github.io/net-ssh/classes/Net/SSH.html#method-c-start
#
# Global options
# --------------
#  set :ssh_options, {
#    keys: %w(/home/rlisowski/.ssh/id_rsa),
#    forward_agent: false,
#    auth_methods: %w(password)
#  }
 
set :ssh_options, {
    forward_agent: true,
    keys: %w(~/.ssh/id_rsa),
    auth_methods: %w(publickey),
    user: 'deployer'
}
 
#
# The server-based syntax can be used to override options:
# ------------------------------------
# server 'example.com',
#   user: 'user_name',
#   roles: %w{web app},
#   ssh_options: {
#     user: 'user_name', # overrides user setting above
#     keys: %w(/home/user_name/.ssh/id_rsa),
#     forward_agent: false,
#     auth_methods: %w(publickey password)
#     # password: 'please use keys'
#   }
```

Your capistrano should be setup for now! However, you need to setup some linked files, e.g. your `config/database.yml` or `config/application.yml` files. Capistrano will not upload these linked files as these files are usually not pushed into your Git as they can contain sensitive information like your API keys and secrets. So we have to copy these files to our application server on the remote server before we can deploy. For this example, we will only copy the database.yml file over. You can repeat the same for other linked files.

First you need to create a folder in your server folder to store all your shared folder items:

```
  $ ssh deployer@xxx.xxx.xxx.xxx

  # in your remote server
  mkdir /home/deployer/apps/app_name/shared/config
```

Enter the commands in your local machine to copy the database.yml, application.yml and secrets.yml for the application.
```
# go into your app folder
$ cd /projects/app_name
$ scp config/database.yml deployer@xxx.xxx.xxx.xxx:/home/deployer/apps/app_name/shared/config
$ scp config/application.yml deployer@xxx.xxx.xxx.xxx:/home/deployer/apps/app_name/shared/config
$ scp config/secrets.yml deployer@xxx.xxx.xxx.xxx:/home/deployer/apps/app_name/shared/config
```

You will then need to edit the database.yml, application.yml and secrets.yml on the server to make sure that they have the right keys and values for the production.

For your database.yml, you need to make sure the following exists for your production:
```
production:
  <<: *default
  database: app_production
  host: localhost
  username: deployer
  password: password_you_entered_for_deployer_postgresql
```

Then run the following in the current folder of your rails application in the server to create the database.
```
  RAILS_ENV=production rake db:create:all
```

For your secrets.yml, make sure to change the secret_key_base for your production part. You can do `rake secret` on your command line console to generate a random secret key you can use.

```
production:
  secret_key_base: ajsdklfjladskfjsdklafjksaldfjasfl;kdsajflkas;fadsf
```  

Voila, its done! Now your linked files and database should be setup, it is time to deploy your application to the server

```
$ cap production deploy
```

Watch and be amazed as you see your application transferring and unwrapping itself through Capistrano!

##If you encounter errors...

You can check the errors of passenger on nginx in the following location:

```
$ cat /opt/nginx/logs/error.log
```

If you encounter an error about Permission Denied when trying to deploy or start passenger, relax permissions on remote server using deployer user:

```
$ sudo chmod g+x,o+x /home/deployer
$ sudo chmod g+x,o+x /home/deployer/apps/
$ sudo chmod g+x,o+x /home/deployer/apps/gameday_api/
$ sudo chmod g+x,o+x /home/deployer/apps/gameday_api/current/
```

If you encounter a circular error where you are unable to install gems during bundle install, it's probably that deployer has no rights to install to the RVM folder. Run the following command for the server:
```
$ chown -R deployer /usr/local/rvm/
```


## References

* https://gorails.com/deploy/ubuntu/14.04/
* http://www.unixmen.com/postgresql-9-4-released-install-centos-7/
* http://vladigleba.com/blog/2014/03/05/deploying-rails-apps-part-1-securing-the-server/
* http://askubuntu.com/questions/192050/how-to-run-sudo-command-with-no-password

## Server Configurations Essentials
* https://www.phusionpassenger.com/library/config/nginx/reference/#passenger_min_instances
* https://www.phusionpassenger.com/documentation/Users%20guide%20Nginx%203.0.html