# Linux Server Configuration

## Project Description

This project is about how to access, secure, and perform the initial configuration of a bare-bones Linux server. And about how to install and configure a web and database server and actually host a web application.

- IP address: 34.221.223.41

- Accessible SSH port: 2200

- Application URL: http://ec2-34-221-223-41.us-west-2.compute.amazonaws.com/

## Walkthrough

### Get A Server

1. Start a new Ubuntu Linux server instance on Amazon Lightsail
- Login into [Amazon Lightsail](https://lightsail.aws.amazon.com/ls/webapp/home/resources) using an Amazon Web Services account.
- Once you are login into the site, click `Create instance`. 
- Choose `Linux/Unix` platform, `OS Only` and  `Ubuntu 16.04 LTS`.
- Choose a instance plan (I choose $3.5/month plan and it's free for the first month!).
- Keep the default name provided by AWS or rename your instance.
- Click the `Create` button to create the instance.
- Wait for the instance to start up.

2. SSH into the server
- From the `Account` menu on Amazon Lightsail, click on `SSH keys` tab and download the Default Private Key.
- Move this private key file named `LightsailDefaultPrivateKey-*.pem` into the local folder `~/.ssh` and rename it `lightsail_key.rsa`.
- In your terminal, type: `chmod 600 ~/.ssh/lightsail_key.rsa`.
- To connect to the instance via the terminal: `ssh -i ~/.ssh/lightsail_key.rsa ubuntu@34.221.223.41`, 
  where `34.221.223.41` is the public IP address of the instance.

3. Create new user named grader and give it the permission to sudo
- Log as root user`$ sudo su -`.
- Then type  `$ sudo adduser grader` to create another user 'grader'.
- Create a new file in the sudoers directory: `$ sudo nano /etc/sudoers.d/grader`. And give grader the super permisssion `grader ALL=(ALL:ALL) ALL`. In nano save with (control X, then type `yes`, then hit the return key on your keyboard).

### Secure The Server

1. Update all currently installed packages and install finger package
- Download package lists with `sudo apt-get update`.
- Fetch new versions of packages with `sudo apt-get upgrade`.
- Install finger package with `$ sudo apt-get install finger`.

2. Change the SSH port from 22 to 2200
- Edit the `/etc/ssh/sshd_config` file: `sudo nano /etc/ssh/sshd_config`.
- Change the port number on line 5 from `22` to `2200`.
- Save and exit using CTRL+X and confirm with Y.
- Restart SSH: `sudo service ssh restart`.

3. Configure the Uncomplicated Firewall (UFW)
- Configure the default firewall for Ubuntu to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
  ```
  sudo ufw status                  
  sudo ufw default deny incoming   
  sudo ufw default allow outgoing  
  sudo ufw allow 2200/tcp          
  sudo ufw allow www               
  sudo ufw allow 123/udp           
  sudo ufw deny 22                 
  ```
- Turn UFW on: `sudo ufw enable`. 
- Check the status of UFW to list current roles: `sudo ufw status`. 
- Exit the SSH connection: `exit`.
- Click on the `Manage` option of the Amazon Lightsail Instance, then the `Networking` tab and find the 'Add another' at the bottom. Add port 123(udp) and 2200(tcp), and deny the default port 22.

4. Use `Fail2Ban` to ban attackers
- Install Fail2Ban: `sudo apt-get install fail2ban`.
- Install sendmail for email notice: `sudo apt-get install sendmail iptables-persistent`.
- Create a copy of a file: `sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`.
- Change the settings in `/etc/fail2ban/jail.local` file:
  ```
  set bantime = 600
  destemail = useremail@domain
  action = %(action_mwl)s 
  ```
- Under `[sshd]` change `port = ssh` by `port = 2200`.
- Restart the service: `sudo service fail2ban restart`.

5. Configure the local timezone to UTC
- Run `sudo dpkg-reconfigure tzdata` and then choose UTC.

6. Configure key-based authentication for grader user
- Open a new terminal window (local machine) and input `$ ssh-keygen -f ~/.ssh/udacity_key.rsa`.
- Read the public key with `$ cat ~/.ssh/udacity_key.rsa.pub` and copy it.
- Return to the first terminal window where you are logged into Amazon Lightsail as the root user, move to grader's folder by `$ cd /home/grader`.
- Create a .ssh directory: `$ mkdir .ssh`
- Create a file to store the public key: `$ touch .ssh/authorized_keys`.
- Run `sudo nano ~/.ssh/authorized_keys` and paste the public key into this file, save and exit.
- Change the permission: `$ sudo chmod 700 /home/grader/.ssh` and `$ sudo chmod 644 /home/grader/.ssh/authorized_keys`.
- Change the owner from root to grader: `$ sudo chown -R grader:grader /home/grader/.ssh`.
- Restart the ssh service: `$ sudo service ssh restart`.
- Type `exit` to disconnect from Amazon Lightsail server.
- Log into the server as grader: `$ ssh -i ~/.ssh/udacity_key.rsa -p 2200 grader@34.221.223.41`(passphrase: grader).

## Deploy Catalog Application

Ensure you are logged in as grader. Should at anypoint a ubuntu password is requested simply ^d and use `sudo` to re-execute that command.

1. Install required packages
- `$ sudo apt-get install apache2`.
- `$ sudo apt-get install libapache2-mod-wsgi python-dev`.
- `$ sudo apt-get install git`.

2. Enable mod_wsgi `$ sudo a2enmod wsgi` and start the web server by `$ sudo service apache2 start`. Enter your public IP address in your browser now and the apache2 default page should be loaded.

3. Create catalog folder to keep app and make grader owner and group of the folder
- `$ cd /var/www`
- `$ sudo mkdir catalog`
- `$ sudo chown -R grader:grader catalog`
- `$ cd catalog`

4. Clone the project from Github: `$ git clone https://github.com/PlumageRan/Udacity-FSND-Item-Catalog.git catalog`.

5. Create a .wsgi file in `/var/www/catalog/`: `$sudo nano catalog.wsgi` (password: grader) and add the following into this file
```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'your_secret_key'
```
6. In /var/www/catalog/catalog Rename the `catalog.py` to `__init__.py` as follows `mv catalog.py __init__.py`.

7. Install virtual environment
- `$ sudo apt-get install python-pip`
- `$ sudo pip install virtualenv`
- `$ sudo virtualenv venv`
- `$ source venv/bin/activate`
- `$ sudo chmod -R 777 venv`

8. Install the Flask and other packages needed for this application
- `$ sudo pip install Flask`
- `$ sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlaclemy_utils requests`

9. Use the `nano __init__.py` command to change the `client_secret_GOOGLE.json` line to `/var/www/catalog/catalog/client_secret.GOOGLE.json` as follows `CLIENT_ID = json.loads(
    open('/var/www/catalog/catalog/client_secret_GOOGLE.json', 'r').read())['web']['client_id']`
	Ensure to look through `__ini__.py` for every instance of this change and replace as stated.
    Also replace
    `if __name__ == '__main__':
    app.secret_key = 'super_secret_key'
    app.debug = True
    app.run(0.0.0.0, port=5000)`

    with

    `if __name__ == '__main__':
    app.secret_key = 'super_secret_key'
    app.debug = True
    app.run()`
	
10. Configure and enable the virtual host
- `$ sudo nano /etc/apache2/sites-available/catalog.conf`
- Paste the following code and save
```
<VirtualHost *:80>
    ServerName 34.221.223.41
    ServerAlias http://ec2-34-221-223-41.us-west-2.compute.amazonaws.com/
    ServerAdmin admin@34.221.223.41
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
You can find your host name in this link: http://www.hcidata.info/host2ip.cgi

11. Set up the database
- `$ sudo apt-get install libpq-dev python-dev`
- `$ sudo apt-get install postgresql postgresql-contrib`
- `$ sudo -u postgres -i`
- Type `$ psql` to get into postgres command line.

12. Create a user to create and set up the database
- `$ CREATE USER catalog WITH PASSWORD password;`
- `$ ALTER USER catalog CREATEDB;`
- `$ CREATE DATABASE catalog WITH OWNER catalog;`
- Connect to database `$ \c catalog`
- `$ REVOKE ALL ON SCHEMA public FROM public;`
- `$ GRANT ALL ON SCHEMA public TO catalog;`
- Quit postgres as follows: `$ \c` and then `$ exit`

13. Use `sudo nano` command to change all engine to `engine = create_engine('postgresql://catalog:password@localhost/catalog)`
Ensure to do this in the database_setup.py, initial_data.py and catalog.py.

14. Initiate the database `sudo python database_setup.py ` and `sudo python initial_data.py `

15. Restart Apache server `$ sudo service apache2 restart` and enter your public IP address or host name into the browser. Your application should be online now.

## Author
Hunter Zhou

## Acknowlegement
- My mentor Tim Nelson for his help and encouragement.
- Useful GitHub Repositories
	- [boisalai/udacity-linux-server-configuration](https://github.com/boisalai/udacity-linux-server-configuration)
	- [juvers/Linux-Configuration](https://github.com/juvers/Linux-Configuration)
	- [rrjoson/udacity-linux-server-configuration](https://github.com/rrjoson/udacity-linux-server-configuration)
