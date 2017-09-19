# How to deploy Ruby on Rails to AWS EC2

*NOTE* This is a work in progress.

In this guide I will describe how to deploy a production Ruby on Rails project to Amazon's EC2 cloud service.

The setup includes the following software:

* Ubuntu Server 14.04 LTS
* RVM (Ruby Version Manager)
* MySql Server (Database)
* Git (Version control)
* Nginx (Reverse proxy server)
* Unicorn (Rails web server)
* Mina (Ruby deployment tool)
* SSL Encryption from letsencrypt.org
* Bitbucket (free private repository hosting)

Practicing RTFM (read the full manual) will help avoid wasting a bunch of time debugging something described in the details. Following this guide should result in flawless deployments but questions and suggestions are always welcome.


## Sections

* Creating the EC2 cloud server
  * Creating the server instance
  * Setting up an Elastic IP address
  * Working with firewall settings (security groups)
  * Creating an SSH alias to quickly log into the server
* Installing the required software
  * [RVM](#RVM)
  * [Git](#Git)
  * [Nodejs](#Nodejs)
  * [MySQL Sßerver](#MySQL-Server)
  * [Nginx](#Nginx)
* Create the MySql Database and User
* Create SSH keys for use with Bitbucket
* Configure Nginx reverse proxy
* Installing a letsencrypt.org SSL certificate
* Deploying with Mina
  * Server settings
  * Local configuration
* Configuring Nginx as a Unicorn reverse proxy

## Creating the EC2 cloud server

### Creating the server instance

Ubuntu 14, small not micro

```
TODO
```

### Setting up an Elastic IP address

```
TODO
```
### Working with firewall settings (security groups)
Update the firewall settings to allow outbound traffic

```
Outbound rule for all traffic to 0.0.0.0/0
TODO
```


### Creating an SSH alias to quickly log into the server
**First a quick warning**

This step is not required. Adding a login alias will expose the key file *and* its associated user/ip-address for the server. If your computer is compromised a person would be one command away from accessing your server. That being said, if your computer is compromised a person could also view your history and gleam the same info (assuming you havn't cleared your history or have it turned off).

Lets begin...

In the Amazon management console create a new authorized key file that will be used strictly for this project.

Download the .pem file and move it to a safe location

```
mv ~/Downloads/rails-project.pem ~/.ec2keys/rails-project.pem
```

Now lets create a shell alias so we can quickly log into the server with a simple command. Name the alias whatever you please. Here I'm using "project-server" but I'm sure you can come up with something better for your needs. Also notice you need to put in the valid ip-address. Refer to [Setting up an Elastic IP address](#Setting up an Elastic IP address) if you havn't done that yet.

```
echo "alias project-server='ssh -i ~/.ec2keys/rails-project.pem ubuntu@put-your-ip-address-here'" >> ~/.bashrc
```

## Installing the required software

### RVM

RVM stands for Ruby Version Manager and provides the ability to easily install multiple Ruby versions. You can read more about it [here](https://rvm.io/rvm/install).

The following command will download and install Ruby and Ruby on Rails via RVM.

```
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB

\curl -sSL https://get.rvm.io | bash -s stable --rails
```

Make sure RVM scripts are reachable by Mina (https://github.com/mina-deploy/mina/issues/5)

```
echo 'source "$HOME/.rvm/scripts/rvm"' >> ~/.bashrc
```

**Important!** Update `.bashrc` to allow RVM in non internactive mode. If you don't do this Bundler and other gems will not be executed by Mina. ((bundle: command not found)[https://github.com/mina-deploy/mina/issues/290#issuecomment-83104437])

**NOTE** This will probably go at the top of the `.bashrc` file

```
# Add RVM to PATH for scripting. Make sure this is the last PATH variable change.
export PATH="$PATH:$HOME/.rvm/bin"
source "$HOME/.rvm/scripts/rvm"

# ^ ^ ^ ^ this must be above the following line.

# If not running interactively, don't do anything
case $- in
    *i*) ;;
      *) return;;
esac
```

### Set the default environment to production

```
# from bash terminal run:

export RAILS_ENV=production
```

### Git

Git is used by Mina during the deployment process to grab code from your repository.

```
sudo apt-get install git
```

### Nodejs

Ruby on Rails requires a server side Javascript executible. Nodejs is the most popular and easiest to install.

```
sudo add-apt-repository ppa:chris-lea/node.js
sudo apt-get -y update
sudo apt-get -y install nodejs
```

### Nginx

Nginx is very fast server technology that is easy to configure and makes a great reverse proxy for our web server.

```
sudo apt-get install nginx
```


### MySQL Server

```
sudo apt-get install mysql-server libmysqlclient-dev
```

The MySql installer will ask your to provide a password for the root database user. Provide a strong password and take note of it. After the installer has completed we will run a handy tool MySql comes with that hardens security.

```
mysql_sercure_installation
```

You shouldn't have a any issues accepting the default answers.

### Create the MySql Database & User

Next we will create the database. Please note the database name, user and user password **do not** need to match what you use in other environments (development, qa).

```
mysql -uroot -p
```

Provide the password you created for the root mysql user.

Once you've logged in to the mysql shell create the database and user.

```
mysql> create database yourdatabase;
mysql> create user 'mysqluser'@'localhost' identified by 'yourpassword';
mysql> grant all privileges on yourdatabase.* to 'mysqluser'@'localhost';
mysql> flush privileges;
mysql> exit
```

### Create SSH keys for use with Bitbucket

An SSH key is required so the deployment tool can be authorized when it attempts to clone the project from the repository. I chose [Bitbucket]() for this tutorial because they offer free private repo hosting. [Gitlab]() also offers free hosting
    ssh-keygen -t rsa

Add id_rsa.pub to bitbucket repository
<put details here>

Set the correct Mysql socket connection for Rails in shared/config/database.yaml. You can find which socket to use by running (http://stackoverflow.com/a/6412218/1491929)

    mysqladmin -uroot -p variables | grep socket

## Unicorn
create required deployment folders

```
mkdir -p shared/tmp/sockets
mkdir -p shared/tmp/pids
```

create config/unicorn.rb

	# set path to application
	app_dir = File.expand_path("../..", __FILE__)
	shared_dir = "#{app_dir}/shared"
	working_directory app_dir

	# Set unicorn options
	worker_processes 2
	preload_app true
	timeout 30

	# Set up socket location
	listen "#{shared_dir}/sockets/unicorn.sock", :backlog => 64

	# Logging
	stderr_path "#{shared_dir}/log/unicorn.stderr.log"
	stdout_path "#{shared_dir}/log/unicorn.stdout.log"

	# Set master PID location
	pid "#{shared_dir}/pids/unicorn.pid"


## Create Unicorn Init Script

(Ripped from https://www.digitalocean.com/community/tutorials/how-to-deploy-a-rails-app-with-unicorn-and-nginx-on-ubuntu-14-04#create-unicorn-init-script)

```
sudo vi /etc/init.d/unicorn_appname
```

Copy and paste the following code block into it, and be sure to substitute USER and APP_NAME (highlighted) with the appropriate values:

```
#!/bin/sh

### BEGIN INIT INFO
# Provides:          unicorn
# Required-Start:    $all
# Required-Stop:     $all
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: starts the unicorn app server
# Description:       starts unicorn using start-stop-daemon
### END INIT INFO

set -e

USAGE="Usage: $0 <start|stop|restart|upgrade|rotate|force-stop>"

# app settings
USER="deploy"
APP_NAME="appname"
APP_ROOT="/home/$USER/www/$APP_NAME/current"
ENV="production"

# environment settings
PATH="/home/$USER/.rbenv/shims:/home/$USER/.rbenv/bin:$PATH"
CMD="cd $APP_ROOT && bundle exec unicorn -c config/unicorn.rb -E $ENV -D"
PID="$APP_ROOT/shared/pids/unicorn.pid"
OLD_PID="$PID.oldbin"

# make sure the app exists
cd $APP_ROOT || exit 1

sig () {
  test -s "$PID" && kill -$1 `cat $PID`
}

oldsig () {
  test -s $OLD_PID && kill -$1 `cat $OLD_PID`
}

case $1 in
  start)
    sig 0 && echo >&2 "Already running" && exit 0
    echo "Starting $APP_NAME"
    su - $USER -c "$CMD"
    ;;
  stop)
    echo "Stopping $APP_NAME"
    sig QUIT && exit 0
    echo >&2 "Not running"
    ;;
  force-stop)
    echo "Force stopping $APP_NAME"
    sig TERM && exit 0
    echo >&2 "Not running"
    ;;
  restart|reload|upgrade)
    sig USR2 && echo "reloaded $APP_NAME" && exit 0
    echo >&2 "Couldn't reload, starting '$CMD' instead"
    $CMD
    ;;
  rotate)
    sig USR1 && echo rotated logs OK && exit 0
    echo >&2 "Couldn't rotate logs" && exit 1
    ;;
  *)
    echo >&2 $USAGE
    exit 1
    ;;
esac
```

Save and exit. This will allow you to use service unicorn_appname to start and stop your Unicorn and your Rails application.

Update the script's permissions and enable Unicorn to start on boot:

```
sudo chmod 755 /etc/init.d/unicorn_appname
sudo update-rc.d unicorn_appname defaults
```

Let's start it now:

```
sudo service unicorn_appname start
```

## Configure the Mina deploy script

generate mina configuration

```
mina init
```

```
# change
invoke :’rails:assets_precompile
# to
invoke :’rails:assets_precompile:force'
```

add unicorn settings to config/deploy.rb

```
set :shared_paths, ['tmp/sockets', 'tmp/pids']
```

when using AWS set option :identity_file and user
set :identity_file, 'keys/key.pem'
set :user ubuntu

when done with configuration run

```
mina setup
```

### Setup shared configuration on the server

```
# add secret key base
shared/config/secrets.yml
```

---
```
****** notes ******
follow this tutorial
https://www.digitalocean.com/community/tutorials/how-to-deploy-a-rails-app-with-unicorn-and-nginx-on-ubuntu-14-04

name unicorn service `unicorn_rails`
USER=ubuntu
APP_NAME=<appname>/current/

DONT FORGET TO ADD .gitignore files to the shared/ folders for unicorn config
****** notes ******

*use mina-unicorn*
mina-nginx
```
---

# Resources and References (and many thanks)
http://www.iredmail.org/docs/use.a.bought.ssl.certificate.html
http://www.iredmail.org/docs/sql.bulk.create.mail.users.html
https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04
https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching
