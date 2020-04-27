# Configure Ubuntu 18.04 has a Production Server running a LEMP stack.

This document will help you configure and prepare a production server using AWS services. 

It is intended for CBDC Restigouche usage, if it happens that you access this file **it's totally fine**, there is no secret here!

*Please note that conventions and example are our owns, adapt to your needs.*

For any question contact regarding this document, feel free to contact [Jonathan Lafleur](mailto:jonathan.lafleur@cbdc.ca).

> # Table of Contents
> 1. [Preparation](#preparation)
> 2. [Install Packages and Dependencies](#packages-dependencies)
> 3. [MySQL](#configure-mysql)
> 4. [Nginx](#nginx)
> 5. [Security](#security)
> 6. [Manage your project with Git](#git)
> 7. [NodeJS and NPM Dependencies](#node)
> 8. [Composer Dependencies](#composer)
> 9. [Install Laravel](#laravel)
> 10. [Wordress](#wordpress)

## Preparation <a name="preparation"></a>

- Go to [AWS Lightsail](https://lightsail.aws.amazon.com/ls/webapp/home)
    - Create instance
        - Make sure the following setting are set:
            - AWS Region is set to Montreal Zone A (ca-central-1a)
            - Linux/Unix
        - Choose OS Only
            - Ubuntu 18.04
        - Make sure that your SSH Key pair is selected
        - Choose your instance plan
            > Be careful, Instance can scale up, but can't scale down. Choose wisely.
        - Identify your instance
            - By Convention we use a 3 letter code for project followed by the service name
                - Exemple: HFT-Services, SIA-Website, etc...
        - Click "**Create instance**"
    - Wait till you get the machine IP and copy it.
- Go to [AWS Route 53](https://console.aws.amazon.com/route53/home?region=ca-central-1)
    - Click on **Hosted Zones**
        - Choose the right domain name
            - From here, you have either a subdomain to create or to change the main A record.
            - Don't forget to create a CNAME record for www that point to your A record you've just created

Voilà! You're up and running, you can now access your new system.

## Install Packages and Dependencies <a name="packages-dependencies"></a>
First you'll need to `ssh` into the machine from your **terminal** or your **putty** app.

By default, the username is `ubuntu`.

First help your "future you" and set the right timezone

```bash
sudo timedatectl set-timezone America/Moncton
```

> If you don't know your timezone, you can list it with:
```bash
timedatectl list-timezones
```


Then we will start updating the packages list then installing the dependencies.
```bash
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install curl git unzip nginx mysql-server php certbot nodejs npm \
    php-{curl,bcmath,bz2,intl,gd,mbstring,mysql,zip,fpm,cli,json,opcache,xml,xsl,interbase,xmlrpc,soap} -y
```

## Configure MySQL <a name="mysql"></a>
```bash
sudo mysql_secure_installation
```

- Validate password plugin : Yes
- Levels of password validation : 2
- Generate a random password for root and save it somewhere...
- Do you wish to continue : Yes
- Remove Anonymous: Yes
- Disallow root remotely: Yes
- Remove Test Database : Yes
- Relaod Privilege : Yes

```bash
sudo mysql -u root
```
```MySQL
CREATE USER '<Username>'@'localhost' IDENTIFIED BY '<Password>';
CREATE DATABASE '<DatabaseName>';
GRANT ALL PRIVILEGES ON <DatabaseName>.* TO '<Username>'@'localhost';
FLUSH PRIVILEGES;
```
> *On a staging system if you want to get rid of mysql root protection:*
 ``` MySQL
UPDATE mysql.user SET plugin = 'mysql_native_password', authentication_string = PASSWORD('<Password>') WHERE User = 'root';
FLUSH PRIVILEGES;
```

Then quit the MySQL CLI (`CTRL+D` or type `exit`)

## Configure Nginx <a name="nginx">

First let's stop Apache
```bash
sudo systemctl stop apache2.service
```

Open your https port (443) on [AWS Lightsail](https://lightsail.aws.amazon.com/ls/webapp/home).


Now go to [nginxconfig.io](https://nginxconfig.io/) and create your configuration files. Follow the whole instruction from [nginxconfig](https://nginxconfig.io/).


Then follow the [nginxconfig.io](https://nginxconfig.io/) steps to install the files.

When done, before testing the URL, type theses command:
```bash
sudo mkdir -p /var/www/domainName/public
sudo echo 'Hello World' >> /var/www/domainName/public/index.php
sudo chown -R www-data /var/www/domainName
```
You are now ready to test your URL.

## Harden installation <a name="security">
To harden your installation, don't forget to always keep a ***working ssh session*** while configuring and testing your modification.
You can lose access to your machine if you don't take care.

1) First let's create a new user

    ```bash
    sudo adduser cbdc
    sudo usermod -aG sudo cbdc
    sudo usermod -aG admin cbdc
    cd /home/cbdc/
    sudo mkdir .ssh
    chmod 700 .ssh
    sudo cp /home/ubuntu/.ssh/authorized_keys ./.ssh/authorized_keys
    chown -R new.username:new.username ./.ssh
    ```
2) Open a new terminal (without closing the first one) and log has the new user, then test if you can `sudo`

    ```bash
    sudo ls -la /root
    ```

   If it work well, close your first session (logged has ubuntu)

3) You can now delete the `ubuntu` user.

    ```bash
    userdel -r ubuntu
    ```
   > if userdel: user ubuntu is currently used by process 1623
   > ```bash
   > kill -9 1623
   > ```

4) Now we will change the default SSH port, to do so, you'll need to modify `ssh_config` file.
    ```bash
    sudo vi /etc/ssh/sshdd_config
   ```
    Modify the following lines
    ```bash
    Port <DesiredSSHPort>
    PermitRootLogin no
    PasswordAuthentication no
    ```
    Then restart the ssh server
    ```bash
    sudo service sshd restart
    ```

    ```bash
    sudo ufw allow <DesiredSSHPort>
    sudo ufw allow http
    sudo ufw allow https
    sudo ufw enable
    ```
    Also go to your [AWS Lightsail](https://lightsail.aws.amazon.com/ls/webapp/home) instance configuration and open the port `TCP <DesiredSSHPort>`.

5) Open a new terminal (without closing the first one) and login to make sure you still can have access to the machine.

6) Secure your shared memory
    ```bash
    sudo vi /etc/fstab
    ```
    Copy at the end of the file
    ```bash
    none /run/shm tmpfs defaults,ro 0 0
    ```

## Git in the project 😉 <a name="git">
- Git should be already installed, if not
  ```bash
  sudo apt-get install git
  ```
- If you not already have a READONLY Git user on CodeCommit, follow theses [instructions](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-unixes.html)
- Install this private key on the server
- Configure SSH agent to use our key for CodeCommit
  ```bash
  cd ~/.ssh/
  vi config
      Host git-codecommit.*.amazonaws.com
          User <AWS-IAMUSER>
          IdentityFile ~/.ssh/<PrivateKeyFile>
  ```
 - You are now ready to `clone` the repo!
 ```bash
git clone ssh://<AWS-IAMUSER>@git-codecommit.<AWS-Region>.amazonaws.com/v1/repos/<RepoName>
```


## Install NodeJS and NPM <a name="node">
```bash
sudo apt-get install nodejs npm -y
```
## Install Composer <a name="composer">
This command will fail in a near future, the hash is for the current version, please update hash accordingly or use the [automated download script](https://getcomposer.org/doc/faqs/how-to-install-composer-programmatically.md)
```bash
cd ~
curl -sS https://getcomposer.org/installer -o composer-setup.php
HASH=e0012edf3e80b6978849f5eff0d4b4e4c79ff1609dd1e613307e16318854d24ae64f26d17af3ef0bf7cfb710ca74755a
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
```
## Configure Laravel <a name="laravel">
```bash
sudo chmod -R YouUsername:www-data <LaravelInstallationPath>
cd <LaravelInstallationPath>
cp .env.example .env
vi .env
```
Configure the .env file **don't forget** to
- APP_ENV=production
- APP_DEBUG=false
- Use Double quote for values that have ambiguous characters or spaces

Now that your .env file is configured, you can start installing Laravel:
```bash
composer install
npm i -y
npm run prod
php artisan storage:link
php artisan key:generate
php artisan migrate
php artisan db:seed
```

## Wordpress <a name="wordpress">
Fix Wordpress file and folder structure:
```bash
cd <WordPressInstallationPath>
wget https://gist.githubusercontent.com/jonathanlaf/0240b8c401651af742ac826a69125440/raw/822eafea11056f55f5a239f8fa7062a43a05d600/fix_permission.sh
chmod +x fix_permission.sh
./fix_permission.sh
```

