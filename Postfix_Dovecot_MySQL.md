# MAIL SERVER
## Install & Config apache2, mysql, php
```
sudo apt-get update
sudo apt-get install apache2
sudo apt-get install mysql-server libapache2-mod-auth-mysql php5-mysql
sudo mysql_install_db
sudo /usr/bin/mysql_secure_installation
```
Answer some question
```
Enter current password for root (enter for none): 
Remove anonymous users? [Y/n] y
Disallow root login remotely? [Y/n] y
Remove test database and access to it? [Y/n] y
Reload privilege tables now? [Y/n] y
```
Install *PHP*
```
sudo apt-get install php5 libapache2-mod-php5 php5-mcrypt
```
Add php to the directory index, to serve the relevant php index files
```
vim /etc/apache2/mods-enabled/dir.conf
<IfModule mod_dir.c>
  DirectoryIndex index.php index.html index.cgi index.pl index.php index.xhtml index.htm
</IfModule>
```
PHP Modules
```
apt-cache search php5-
sudo apt-get install name of the module
sudo service apache2 restart
```

## Get certificate SSL
-> go to startssl.com -> login -> verify domain
```
openssl req -newkey rsa:2048 -keyout yourname.key -out yourname.csr
```
-> submit CSR -> get Cert file.
-> upload to server.


## Install & Config ***Mail Server***
### Step1: Install packages
```
vim /etc/hostname
mail.hiepnguyen.xyz
sudo reboot
```
```sh
sudo -i
apt-get install postfix postfix-mysql dovecot-core dovecot-imapd dovecot-lmtpd dovecot-mysql
```
```
SSL Sign: yes
Hostname: mail
Internet Site
FDN: mail.hiepnguyen.xyz
```
### Step 2: Create a MySQL Database, Virtual Domains, Users and Aliases
Create DB
```sh
mysqladmin -p create servermail
mysql -u root -p
mysql>
```
Grant & create table on mysql CLI
```
GRANT SELECT ON servermail.* TO 'hiepnguyen'@'127.0.0.1' IDENTIFIED BY '123456';
FLUSH PRIVILEGES;
USE servermail;

CREATE TABLE `virtual_domains` (
    `id`  INT NOT NULL AUTO_INCREMENT, 
    `name` VARCHAR(50) NOT NULL, 
    PRIMARY KEY (`id`)) 
    ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `virtual_users` (
    `id` INT NOT NULL AUTO_INCREMENT, 
    `domain_id` INT NOT NULL, 
    `password` VARCHAR(106) NOT NULL, 
    `email` VARCHAR(120) NOT NULL, 
    PRIMARY KEY (`id`), 
    UNIQUE KEY `email` (`email`), 
    FOREIGN KEY (domain_id) 
    REFERENCES virtual_domains(id) ON DELETE CASCADE ) 
    ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `virtual_aliases` ( 
    `id` INT NOT NULL AUTO_INCREMENT, 
    `domain_id` INT NOT NULL, 
    `source` varchar(100) NOT NULL, 
    `destination` varchar(100) NOT NULL, 
    PRIMARY KEY (`id`), 
    FOREIGN KEY (domain_id) 
    REFERENCES virtual_domains(id) ON DELETE CASCADE) 
    ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
Insert Value
```
INSERT INTO `servermail`.`virtual_domains`
(`id` ,`name`)
VALUES
('1', 'hiepnguyen.xyz'),
('2', 'mail.hiepnguyen.xyz');

INSERT INTO `servermail`.`virtual_users`
(`id`, `domain_id`, `password` , `email`)
VALUES
('1', '1', ENCRYPT('123456', CONCAT('$6$', SUBSTRING(SHA(RAND()), -16))), 'bobby1@hiepnguyen.xyz'),
('2', '1', ENCRYPT('123456', CONCAT('$6$', SUBSTRING(SHA(RAND()), -16))), 'bobby2@hiepnguyen.xyz');

INSERT INTO `servermail`.`virtual_aliases`
(`id`, `domain_id`, `source`, `destination`)
VALUES
('1', '1', 'support@hiepnguyen.xyz', 'bobby2@hiepnguyen.xyz');
```

### Step 3: Configure Postfix
```sh
cp /etc/postfix/main.cf /etc/postfix/main.cf.orig
vim /etc/postfix/main.cf
```
```sh
# TLS parameters
#smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
#smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
#smtpd_use_tls=yes
#smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
#smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache 
smtpd_tls_cert_file=/etc/dovecot/dovecot.pem
smtpd_tls_key_file=/etc/dovecot/private/dovecot.pem
smtpd_use_tls=yes
smtpd_tls_auth_only = yes

smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_recipient_restrictions =
        permit_sasl_authenticated,
        permit_mynetworks,
        reject_unauth_destination

myhostname = mail.hiepnguyen.xyz
#mydestination = hiepnguyen.xyz, etestme.c.etestme-144311.internal, mailocalhost.c.etestme-144311.internal, localhost
mydestination = localhost

## Tells Postfix to use Dovecot's LMTP instead of its own LDA to save emails to the local mailboxes.
virtual_transport = lmtp:unix:private/dovecot-lmtp

## Tells Postfix you're using MySQL to store virtual domains, and gives the paths to the database connections.
virtual_mailbox_domains = mysql:/etc/postfix/mysql-virtual-mailbox-domains.cf
virtual_mailbox_maps = mysql:/etc/postfix/mysql-virtual-mailbox-maps.cf
virtual_alias_maps = mysql:/etc/postfix/mysql-virtual-alias-maps.cf
```
Compare these changes with [this file](https://www.dropbox.com/s/x9fpm9v1dr86gkw/etc-postfix-main.cf.txt) to detect mistakes or errors.

Create 3 files
```sh
vim /etc/postfix/mysql-virtual-mailbox-domains.cf
```
```
user = hiepnguyen
password = 123456
hosts = 127.0.0.1
dbname = servermail
query = SELECT 1 FROM virtual_domains WHERE name='%s'
```
```sh
service postfix restart
postmap -q hiepnguyen.xyz mysql:/etc/postfix/mysql-virtual-mailbox-domains.cf
1
```

```sh
vim /etc/postfix/mysql-virtual-mailbox-maps.cf 
```
```
user = hiepnguyen
password = 123456
hosts = 127.0.0.1
dbname = servermail
query = SELECT 1 FROM virtual_users WHERE email='%s'
```

```sh
service postfix restart
postmap -q bobby1@hiepnguyen.xyz mysql:/etc/postfix/mysql-virtual-mailbox-maps.cf
1
postmap -q bobby2@hiepnguyen.xyz mysql:/etc/postfix/mysql-virtual-mailbox-maps.cf
1
```

```sh
vim /etc/postfix/mysql-virtual-alias-maps.cf
user = hiepnguyen
password = 123456
hosts = 127.0.0.1
dbname = servermail
query = SELECT destination FROM virtual_aliases WHERE source='%s'
```

```sh
service postfix restart
postmap -q support@hiepnguyen.xyz mysql:/etc/postfix/mysql-virtual-alias-maps.cf
bobby2@hiepnguyen.xyz
```

```sh
vim /etc/postfix/master.cf
submission inet n       -       -       -       -       smtpd
-o syslog_name=postfix/submission
-o smtpd_tls_security_level=encrypt
-o smtpd_sasl_auth_enable=yes
-o smtpd_client_restrictions=permit_sasl_authenticated,reject
```
```
service postfix restart
```
**Note**: You can use [this tool](http://mxtoolbox.com/SuperTool.aspx) to scan your domain ports and verify that port 25 and 587 are open.

### Step 4: Configure Dovecot
**1,** Backup 7 files
```
cp /etc/dovecot/dovecot.conf /etc/dovecot/dovecot.conf.orig
cp /etc/dovecot/conf.d/10-mail.conf /etc/dovecot/conf.d/10-mail.conf.orig
cp /etc/dovecot/conf.d/10-auth.conf /etc/dovecot/conf.d/10-auth.conf.orig
cp /etc/dovecot/dovecot-sql.conf.ext /etc/dovecot/dovecot-sql.conf.ext.orig
cp /etc/dovecot/conf.d/10-master.conf /etc/dovecot/conf.d/10-master.conf.orig
cp /etc/dovecot/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf.orig
```
**2,** Edit configuration file from Dovecot.
```sh
vim /etc/dovecot/dovecot.conf
```
Verify this option is uncommented.
```sh
!include conf.d/*.conf
```
We are going to enable protocols (add pop3 if you want to) below the `!include_try /usr/share/dovecot/protocols.d/*.protocol` line.
```
!include_try /usr/share/dovecot/protocols.d/*.protocol
protocols = imap lmtp
```

**Note**: Compare these changes with [this file ](https://www.dropbox.com/s/wmbe3bwy0vcficj/etc-dovecot-dovecot.conf.txt) to detect mistakes or errors.

**3,** Edit the *mail configuration* file:
```sh
vim /etc/dovecot/conf.d/10-mail.conf
```

Find the *mail_location* line, **uncomment** it, and **put** the following parameter:
```
mail_location = maildir:/var/mail/vhosts/%d/%n
```
Find the *mail_privileged_group* line, **uncomment** it, and **add** the mail parameter like so:
```
mail_privileged_group = mail
```
**Note**: Compare these changes with [this file](https://www.dropbox.com/s/hnfeieuy77m5b0a/etc.dovecot.conf.d-10-mail.conf.txt) to detect mistakes or errors

**4,** Verify permissions
Ensure
```
ls -ld /var/mail
drwxrwsr-x 3 root vmail 4096 Jan 24 21:23 /var/mail
```
We are going to *create a folder for each domain* that we register *in the MySQL table*:
```
mkdir -p /var/mail/vhosts/hiepnguyen.xyz
mkdir -p /var/mail/vhosts/mail.hiepnguyen.xyz
```
Create a vmail user and group with an id of 5000. Change the owner of the /var/mail folder to the vmail user
```sh
groupadd -g 5000 vmail 
useradd -g vmail -u 5000 vmail -d /var/mail
chown -R vmail:vmail /var/mail
```
Edit 10-auth.conf
```sh
vim /etc/dovecot/conf.d/10-auth.conf
```
Uncomment plain text authentication and add this line:
```
disable_plaintext_auth = yes
```
Modify `auth_mechanisms` parameter:
```
auth_mechanisms = plain login
```
Comment this line:
```
#!include auth-system.conf.ext
```
Enable MySQL authorization by uncommenting this line:
```
!include auth-sql.conf.ext
```
**Note:** Compare these changes with [this file](https://www.dropbox.com/s/4h472nqrj700pqk/etc.dovecot.conf.d.10-auth.conf.txt) to detect mistakes or errors.

Create dovecot-sql.conf.ext file and Edit auth-sql.conf.ext file
```
vim /etc/dovecot/conf.d/auth-sql.conf.ext
passdb {
  driver = sql
  args = /etc/dovecot/dovecot-sql.conf.ext
}
userdb {
  driver = static
  args = uid=vmail gid=vmail home=/var/mail/vhosts/%d/%n
} 
```
Modify the dovecot-sql.conf.ext file with our custom MySQL information
```sh
# vim /etc/dovecot/dovecot-sql.conf.ext
```
Uncomment the driver parameter and set mysql as parameter:
```
driver = mysql
```
Uncomment the connect line and introduce your *MySQL specific information*:
```
connect = host=127.0.0.1 dbname=servermail user=hiepnguyen password=123456
```
Uncomment the *default_pass_scheme* line and change it to SHA-512.
```
default_pass_scheme = SHA512-CRYPT
```
Uncomment the *password_query* line and add this information:
```
password_query = SELECT email as user, password FROM virtual_users WHERE email='%u';
```
**Note:** Compare these changes with [this file](https://www.dropbox.com/s/48a5r0mtgdz25cz/etc.dovecot.dovecot-sql.conf.ext.txt) to detect mistakes or errors
Change the owner and the group of the dovecot folder to vmail user:
```sh
chown -R vmail:dovecot /etc/dovecot
chmod -R o-rwx /etc/dovecot 
```

Open and modify the `/etc/dovecot/conf.d/10-master.conf` file (be careful because different parameters will be changed).
```sh
vim /etc/dovecot/conf.d/10-master.conf
```
```
##Uncomment inet_listener_imap and modify to port 0
service imap-login {
  inet_listener imap {
    port = 0
}

#Create LMTP socket and this configurations
service lmtp {
   unix_listener /var/spool/postfix/private/dovecot-lmtp {
       mode = 0600
       user = postfix
       group = postfix
   }
  #inet_listener lmtp {
    # Avoid making LMTP visible for the entire internet
    #address =
    #port =
  #}
} 
```

Modify `unix_listener` parameter to service_auth like this:

```
service auth {

  unix_listener /var/spool/postfix/private/auth {
  mode = 0666
  user = postfix
  group = postfix
  }

  unix_listener auth-userdb {
  mode = 0600
  user = vmail
  #group =
  }

  #unix_listener /var/spool/postfix/private/auth {
  # mode = 0666
  #}

  user = dovecot
}
```

Modify `service auth-worker` like this:

```
service auth-worker {
  # Auth worker process is run as root by default, so that it can access
  # /etc/shadow. If this isn't necessary, the user should be changed to
  # $default_internal_user.
  user = vmail
}
```

**Note**: Compare these changes with [this file](https://www.dropbox.com/s/g0vnt233obh6v2h/etc.dovecot.conf.d.10-master.conf.txt) to detect mistakes or errors.

Finally, we are going to *modify the SSL configuration file from Dovecot* (skip this step if you are going to use default configuration).
```sh
vim /etc/dovecot/conf.d/10-ssl.conf
```
Change the ssl parameter to required:
```
ssl = required
```

And modify the path for `ssl_cert` and `ssl_key`:

```
ssl_cert = </etc/dovecot/dovecot.pem
ssl_key = </etc/dovecot/private/dovecot.pem
```

Restart Dovecot
```sh
service dovecot restart
```
You should check that port 993 is open and working (in case you enable pop3; you should check also port 995).
```sh
telnet hiepnguyen.xyz 993
```
**Congratulations.** You have successfully configured your mail server and you may test your account using an email client:
```
- Username: bobby1@hiepnguyen.xyz
- Password: 123456
- IMAP: hiepnguyen.xyz
- SMTP: hiepnguyen.xyz
```