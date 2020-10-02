---
title:  "Installing Gitea on Ubuntu"
date:   2020-10-01 16:30:00
categories: [gitea, ubuntu, nginx, letsencrypt]
tags: [gitea, ubuntu, nginx, letsencrypt]
---

Through collaboration the need for a private/self-hosted GIT installation arose. After some looking around, I ended up choosing Gitea as it aims to be a painless way for setting up a self-hosted Git service. It is written in Go and distributed as a binary that is cross-platform. I came across several guides to ease the installation process and ended up using some bits of each and some additions of my own. Let's dive right into it.

### Add Git user and Insatll Git

We need to add a git system user that gitea will use. This user will have login's disabled and its system shell set to git-shell. Before adding the user, we must first make git-shell a valid system shell by adding it to /etc/shells.

#### Add git-shell to /etc/shells

```
sudo echo "/usr/bin/git-shell" >> /etc/shells
```

#### Create git user

Most of the guides I read online set the shell to /bin/bash even though they were disabling passwords. I wanted to restrict this even further by setting it to git-shell. Which is why the first step was to add git-shell to /etc/shells.

{% highlight console %}
sudo adduser --system --shell /usr/bin/git-shell --gecos 'Git Version Control' --group --disabled-password --home /home/git git
{% endhighlight %}

#### Install Git

At this point we can install the git service using apt repositories. Some server installations will have this step completed but we will include in our steps anyway.

{% highlight console %}
sudo apt update
sudo apt install -y git
{% endhighlight %}

Now we can begin the process of installing and configuring MariaDB for use with Gitea.

### Install and Configure MariaDB

Gitea requires a backend database server to store its content (users, repos, code, etc..). MariaDB seems to be a nice fit for our requirements and easily installed and configured. We can quickly install it via apt.

#### Install MariaDB from the command line


```
sudo apt install -y mariadb-server mariadb-client
```

Once complete we want to secure our mysql installation by running mysql_secure_installation. This will address the following (among others):

<ol>
<li>Entering current root password (if set)</li>
<li>Setting root password</li>
<li>Remove Anonymous users</li>
<li>Disallow root login remotely</li>
<li>Remove test database and access to it</li>
<li>Reloading privilege table</li>
</ol>

#### Run mysql_secure_installation

```
sudo mysql_secure_installation
```

Answer the questions for each prompt, then we will proceed to the actual configuration of the database Gitea will use.

#### Creating gitea database

Now that the required packages for gitea to run are installed (git and mariadb) we can proceed to configure the database and then install the actual gitea binary to get the system up and running. Let's login to mysql:

{% highlight console %}
mysql -u root -p
{% endhighlight %}

Once you login using the credentials we set in the previous section, lets create the database that gitea will use.

{% highlight sql %}
CREATE DATABASE gitea;
{% endhighlight %}

Now we need to create a database user followed by granting that user the necessary permissions to the "gitea" database.

{% highlight sql %}
CREATE USER 'gitea'@'localhost' IDENTIFIED BY 'new_password_here';
{% endhighlight %}

Now we grant ALL necessary permissions for user "gitea" to database "gitea"

{% highlight sql %}
GRANT ALL ON gitea.* TO 'gitea'@'localhost' IDENTIFIED BY 'user_password_here' WITH GRANT OPTION;
{% endhighlight %}

The final step is to change the character set to utf8mb4 with the following command.

{% highlight sql %}
ALTER DATABASE gitea CHARACTER SET = utf8mb4 COLLATE utf8mb4_unicode_ci;
{% endhighlight %}

Now we will flush privileges to save our changes and exit.

{% highlight sql %}
FLUSH PRIVILEGES;
EXIT;
{% endhighlight %}


Before moving forward with setting up the gitea file-system environment, we must make a change to the mariadb configuration file to instruct it to use the InnoDB storage engine since this is required by newer versions of Gitea. We need to edit

```
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Add the following lines to the config file, save and exit.

{% highlight console %}
innodb_file_format = Barracuda
innodb_large_prefix = 1
innodb_default_row_format = dynamic
{% endhighlight %}

Ctrl+O then Ctrl+X to save and quit respectively.



### Gitea Installation

We are now ready to get the Gitea file-system environment set up with necessary directories and permissions. I have automated this process with a simple shell script that you can find at the end of this section, but for the guide, I will illustrate each command.

#### Download Gitea binary

```
cd /tmp
wget https://dl.gitea.io/gitea/1.12.4/gitea-1.12.4-linux-amd64
```

Now we must move the binary to its final spot (/usr/local/bin/gitea) and make it executable.

{% highlight console %}
sudo mv gitea-1.12.4-linux-amd64 /usr/local/bin/gitea
sudo chmod +x /usr/local/bin/gitea
{% endhighlight %}

#### Gitea fs environment

Now that we have "installed" the gitea binary, we need to create the relevant directories used by the service and set the permissions accordingly.

```
sudo mkdir -p /var/lib/gitea/{custom,data,indexers,public,log}
sudo chown git:git /var/lib/gitea/{data,indexers,log}
sudo chmod 750 /var/lib/gitea/{data,indexers,log}
sudo mkdir /etc/gitea
sudo chown root:git /etc/gitea
sudo chmod 770 /etc/gitea
```

Let's break down what is happening here.

  1. Create the necessary directories {custom,data,indexers,public,log} within /var/lib/gitea. The -p switch will create the parent directory first if it doesn't exist {/var/lib/gitea} and the sub-directories as well {custom,data,indexers,public,log).
  2. The chown (change ownership)  sets user:group to git:git for the {data,indexers,log} directories.
  3. Set permissions to the {data,indexers,log} directories so that Owner can read/write/execute. Group can read and execute but not write. Others have no access at all.
  4. Create /etc/gitea direcotry
  5. chown (change ownership) to user root group git for directory /etc/gitea
  6. Set permissions for /etc/gitea that Owner and Group can read/write/execute and Others have no access at all.

Now we need to set up a systemd config file to enable gitea at boot and to use systemctl to start/stop/status the gitea web service.

#### Create gitea.service file

Now we need to create /etc/systemd/system/gitea.service and then reload systemd daemon and see if we are able to stop/start gitea.

{% highlight console %}
sudo nano /etc/systemd/system/gitea.service
[Unit]
Description=Gitea (Git with a cup of tea)
After=syslog.target
After=network.target
#After=mysqld.service
#After=postgresql.service
#After=memcached.service
#After=redis.service

[Service]
# Modify these two values and uncomment them if you have
# repos with lots of files and get an HTTP error 500 because
# of that
###
#LimitMEMLOCK=infinity
#LimitNOFILE=65535
RestartSec=2s
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
ExecStart=/usr/local/bin/gitea web -c /etc/gitea/app.ini
Restart=always
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea
# If you want to bind Gitea to a port below 1024 uncomment
# the two values below
###
#CapabilityBoundingSet=CAP_NET_BIND_SERVICE
#AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target

{% endhighlight %}

Now we must reload systemd daemon, enable gitea at boot, start it manually and then check its status to verify everything.

{% highlight console %}
sudo systemctl daemon-reload
sudo systemctl enable gitea
sudo systemctl start gitea
sudo systemctl status gitea
{% endhighlight %}

You should see something similar when invoking `sudo systemctl status gitea`

{% highlight console %}
electr0n@vmi421940:~$ sudo systemctl status gitea
● gitea.service - Gitea (Git with a cup of tea)
     Loaded: loaded (/etc/systemd/system/gitea.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2020-10-02 06:26:35 CEST; 2h 21min ago
   Main PID: 683296 (gitea)
      Tasks: 13 (limit: 19149)
     Memory: 156.1M
     CGroup: /system.slice/gitea.service
             └─683296 /usr/local/bin/gitea web -c /etc/gitea/app.ini
{% endhighlight %}

At this point, the system is ready to begin the web installation piece by pointing your browser to http://domain:3000/install but we are going to take this a step further by using nginx as a reverse proxy with LetsEncrypt to force HTTPS even when trying HTTP. Let's get it!!

### Nginx / LetsEncrypt

To continue we must first install both nginx and letsencrypt with the following command

```
sudo apt install -y nginx certbot python3-certbox-nginx
```

Now we need to make sure nginx is not running by issuing stop to service command.

```
sudo service nginx stop
```

#### Create SSL cert

Run we are ready to run certbot (LetsEncrypt) to issue a standalone SSL certificate to use with our implementation of Gitea.  

```
sudo certbot certonly --standalone -d git.cerveau.us
```

Upon successful completion our SSl certificate and key will be in the /etc/letsencrypt/live/git.cerveau.us directory.

1. fullchain.pem = SSL certificate

2. privkey.pem = SSL private key

#### Config Nginx to use Certs

Now we must instruct nginx where to find these files and setup the reverse proxy so we can lose :3000 in our URL. Using nano we need to 'create' /etc/nginx/sites-available/git.cerveau.us with the following content:

{% highlight nginx %}
server {
    listen 443 ssl;
    server_name git.cerveau.us;
    ssl_certificate /etc/letsencrypt/live/git.cerveau.us/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/git.cerveau.us/privkey.pem;

    location / {
        proxy_set_header  X-Real-IP  $remote_addr;
        proxy_pass http://localhost:3000;
    }
}

# Redirect HTTP requests to HTTPS
server {
    listen 80;
    server_name git.cerveau.us;
    return 301 https://$host$request_uri;
}
{% endhighlight %}

The 'server' section instructs Nginx to listen on SSL port 443. Where on the file-system the SSL certificate and private key are for this corresponding domain. The second config section is redirecting / (default GET request) to localhost port 3000 (gitea web service). The final section is enforcing the use of HTTPS using a 301 (redirect). This allows us to only use https://git.cerveau.us in our browser vs. https://git.cerveau.us:3000

#### Enable site

Now we create a symbolic link (ln -s} from sites-available to sites-enabled as follows:

```
sudo ln -s /etc/nginx/sites-available/git.cerveau.us /etc/nginx/sites-enabled
```

Now we can start the nginx service for these changes to take affect.

```
sudo service nginx starting
```

Now you are ready to point your browser to https://<yourdomain>/install to begin the installation process of the gitea web service.

### Gitea install

This portion is straight-forward, make sure the db user / pass match accordingly. Setup an admin user for the site and finish it off by clicking on "Install Gitea".

#### Gitea post-install

After the initial install, I went ahead and locked down the installation a bit further by making the necessary changes to /etc/gitea/app.ini. I modified the following values:

```
1. DISABLE_REGISTRATION to true //disable user registration
2. ENABLE_OPENID_SIGNUP to false //disallow openid signup
3. ENABLE_OPENID_SIGNIN to false //disallow openid signin.
4. REQUIRE_SIGNIN_VIEW to true //require sign in to view files
5. DOMAIN to git.cerveau.us // yours will vary
6. ENABLE_LETSENCRYPT to true //Make sure DOMAIN matches name in ssl_certificate
7. SHOW_REGISTRATION_BUTTON to false // do not show button since we disabled registration.
8. PASSWORD_COMPLEXITY to lower,upper,digit,spec // Must include lowercase, uppercase, digits and special characters.
```

Some other configuration options to look at for future consideration are:

```
1. ENABLE_BASIC_AUTHENTICATION
2. ENABLE_REVERSE_PROXY_AUTHENTICATION
3. ENABLE_CAPTCHA
```

For an entire configuration [cheat-sheet for Gitea](https://docs.gitea.io/en-us/config-cheat-sheet/). There are a few other options I may look into enabling or disabling but for now, the system is at a point where I am comfortable creating repos and pushing code to the site. For a small added bonus, a small automation script to tie it all together.


{% highlight console %}
#!/bin/bash

# Gitea download and configuration, make sure to run script with sudo



# add git-shell to /etc/shells
sudo echo "/usr/bin/git-shell" >> /etc/shells

# create git user

echo "Creating git user"
sudo adduser --system --shell /usr/bin/git-shell --gecos 'Git Version Control' --group --disabled-password --home /home/git git

# Install all necessary packages
echo "Installing git/mariadb/nginx/certbox"
sudo apt install -y git mariadb-server mariadb-client nginx certbox python3-certbot-nginx

## Create database, user, grant permissions

echo "Enter root password for mysql"
read mysql_rootpassword

echo "Enter name for gitea database"
read gitea_database

mysql -u root -p $mysql_rootpassword -e "CREATE DATABASE $gitea_database;"

echo "Creatied gitea database: $gitea_database\n"

echo "Enter username for gitea database"
read gitea_username

echo "Enter password for gitea database user"
read gitea_password


echo "Creating gitea db user: $gitea_username with specified password"

mysql -u root -p $mysql_rootpassword -e "CREATE USER '$gitea_username'@'localhost' IDENTIFIED BY '$gitea_password';"

mysql -u root -p $mysql_rootpassword -e "GRANT ALL ON $gitea_database.* TO '$gitea_username'@'localhost' IDENTIFIED BY '$gitea_password' WITH GRANT OPTION;"
mysql -u root -p $mysql_rootpassword -e "FLUSH PRIVILEGES;"

## get latest version of gitea

echo "Downloading gitea binary"
cd /tmp
wget https://dl.gitea.io/gitea/1.12.4/gitea-1.12.4-linux-amd64

## move binary to /usr/local/bin ahd make executable.

echo "Installing binary to /usr/local/bin/gitea and marking it executable"
sudo mv gitea-1.12.4-linux-amd64 /usr/local/bin/gitea
sudo chmod +x /usr/local/bin/gitea

## create necessary directories and change permissions accordingly

echo "Creating directories for gitea and setting permissions"

sudo mkdir -p /var/lib/gitea/{custom,data,indexers,public,log}
sudo chown git:git /var/lib/gitea/{data,indexers,log}
sudo chmod 750 /var/lib/gitea/{data,indexers,log}
sudo mkdir /etc/gitea
sudo chown root:git /etc/gitea
sudo chmod 770 /etc/gitea

## Create gitea systemd script

echo "Creating gitea.service systemd file"

sudo cat >> /etc/systemd/system/gitea.service <<EOL
[Unit]
Description=Gitea (Git with a cup of tea)
After=syslog.target
After=network.target
#After=mysqld.service
#After=postgresql.service
#After=memcached.service
#After=redis.service

[Service]
# Modify these two values and uncomment them if you have
# repos with lots of files and get an HTTP error 500 because
# of that
###
#LimitMEMLOCK=infinity
#LimitNOFILE=65535
RestartSec=2s
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
ExecStart=/usr/local/bin/gitea web -c /etc/gitea/app.ini
Restart=always
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea
# If you want to bind Gitea to a port below 1024 uncomment
# the two values below
###
#CapabilityBoundingSet=CAP_NET_BIND_SERVICE
#AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
EOL

## reload systemd daemon, enable gitea at boot and start it now.

echo "reload systemctl daemon, enable gitea at boot, start gitea"

systemctl daemon-reload
systemctl enable gitea
systemctl start gitea

### Create SSL certbot

echo "Enter domain name for SSL cert"
read domain_ssl

sudo certbot certonly --standalone -d $domain_ssl

# Configure Nginx as a reverse proxy and to use the SSL certbot


echo "Creating nginx config file, using $domain_ssl from previous entry for site"

sudo cat >> /etc/nginx/sites-available/$domain_ssl << EOL

server {
    listen 443 ssl;
    server_name $domain_ssl;
    ssl_certificate /etc/letsencrypt/live/$domain_ssl/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/$domain_ssl/privkey.pem;

    location / {
        proxy_set_header  X-Real-IP  $remote_addr;
        proxy_pass http://localhost:3000;
    }
}

# Redirect HTTP requests to HTTPS
server {
    listen 80;
    server_name $domain_ssl;
    return 301 https://$host$request_uri;
}
EOL

echo "Enabling site with symbolic link"

sudo ln -s /etc/nginx/sites-available/$domain_ssl /etc/nginx/sites-enabled/

echo "Reloading Nginx for changes to take affect"

systemctl restart nginx

{% endhighlight %}
