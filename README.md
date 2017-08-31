# Linux Server Configuration project for the Udacity Full Stack Developer Nanodegree.

The AWS Lightsail Server instance can be found with the following:
IP address         : 35.158.51.72
SSH Port           : 2200
Web Application URL: http://ec2-35-158-51-72.eu-central-1.compute.amazonaws.com/

## Summary of Configuration Steps
1. Used the Amazon Management Console to create a custom Lightsail Firewall rule to allow connections on Port 2200.
2. Changed SSH port from 22 to 2200:
   $ sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config_backup
   $ sudo nano /etc/ssh/sshd_config
   Changed Port 22 to Port 2200
   $ sudo service sshd restart
   (https://www.liquidweb.com/kb/changing-the-ssh-port/)
3. Used the Uncomplicated Firewall (UFW) tool to configure the firewall:
   $ sudo ufw default deny incoming
   $ sudo ufw default allow outgoing
   $ sudo ufw allow www
   $ sudo ufw allow 2200
   $ sudo ufw allow ntp
   $ sudo ufw enable
   $ sudo ufw status
4. Gave grader access to the server:
    * Created _grader_ account:
        $ sudo adduser grader
    * Added _grader_ to list of sudoers by creating a _grader_ file in _/etc/sudoers.d/_ with the following line:
        grader ALL=(ALL) NOPASSWD:ALL
    * Generated public/private key pair for grader on local machine.
        $ ssh-keygen
    * Created an authorized key file for _grader_
        $ cd /home/grader
        $ su - grader
        $ mkdir .ssh
        $ touch .ssh/authorized_keys
        $ chmod 700 .ssh
        $ chmod 644 .ssh/authorized_keys
    * Installed public key onto the server by pasting the key into the _/home/grader/.ssh/authorized_keys_ file.
5. Installed Apache and mod_wsgi:
    $ sudo apt-get install apache2
    $ sudo apt-get install libapache2-mod-wsgi
6. Configured Apache by editing _/etc/apache2/sites-enabled/000-default.conf_
    * Added a WSGIScriptAlias for the application:
        WSGIScriptAlias / /var/www/itemcatalog/fsnd-item-catalog/itemcatalog.wsgi
        &lt;Directory "/var/www/itemcatalog/fsnd-item-catalog"&gt; Require all granted
        &lt;/Directory&gt;
        (https://httpd.apache.org/docs/2.4/configuring.html)
        (https://code.google.com/archive/p/modwsgi/wikis/ConfigurationGuidelines.wiki)
    * Added aliases for static files:
        Alias "/static/" "/var/www/itemcatalog/fsnd-item-catalog/static/"
        Alias "/templates/" "/var/www/itemcatalog/fsnd-item-catalog/templates/"
        (https://httpd.apache.org/docs/2.4/mod/mod_alias.html#alias)
    * Restart Apache:
        $ sudo apache2ctl restart
7. Setup PostgreSQL:
    * Installed postgresql and it's Python DB-API:
        $ sudo apt-get install postgresql
        $ sudo apt-get install python-psycopg2
    * Changed the postgres account password:
        $ sudo -u postgres psql
            postgres=# \password postgres
    * Created the Item Catalog user:
        postgres=# CREATE USER catalog WITH PASSWORD '_password_';
    * Created the Item Catalog database:
        postgres=# CREATE DATABASE catalogdb;
        postgres=# \c catalogdb;
    * Grant user privileges on the database:
        postgres=# GRANT ALL PRIVILEGES ON DATABASE catalogdb TO catalog;
    * Configured PostGreSQL to use md5 authentication by editing _/etc/postgresql/9.5/main/pg_hba.conf_:
        `local   all    postgres    md5`
        `local   all    all         md5`
    * Restart PostgreSQL:
        $ sudo service postgresql restart
    (http://postgresguide.com/setup/install.html)
8. Installed the pip Python package management system:
    $ sudo apt-get install python-pip
9. Installed Python packages required by the Item Catlog web application:
    $ sudo pip install requests
    $ sudo pip install Flask
    $ sudo pip install sqlalchemy
    $ sudo pip install --upgrade oauth2client
10. Installed and configured git:
    $ sudo apt-get install git
    $ git config --global user.name "_username_"
    $ git config --global user.email "_emailAddress_"
    (https://www.atlassian.com/git/tutorials/install-git#linux)
11. Deployed the Item Catalog web application project:
    * Cloned the git repository:
        $ cd /var/www
        $ mkdir itemcatalog
        $ cd itemcatalog
        $ git clone https://github.com/MickElliott/fsnd-item-catalog.git
12. Adapted the Item Catalog to work as a WSGI application:
    * Changed the call to the create_engine() function of SQLAlchemy to use the PostgreSQL database.
        `engine = create_engine('postgresql+psycopg2://catalog:password@localhost/catalogdb')`
        (http://docs.sqlalchemy.org/en/latest/core/engines.html)
    * Changed the Flask run() function to use Port 80 instead of 5000:
        `app.run(host='0.0.0.0', port=80)`
    * Created a WSGI script, called itemcatalog.wsgi, to launch the application:
        `import sys`
        `sys.path.insert(0, '/var/www/itemcatalog/fsnd-item-catalog')`
        `from application import app as application`
        (http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/)
13. Used the existing Python scripts from the Item Catalog repository to setup the database and load it with sample data:
    $ cd /var/www/itemcatalog/fsnd-item-catalog
    $ python database_setup.py
    $ python lotsofcategories.py
14. Upgraded the installed packages:
    $ sudo apt-get update
    $ sudo apt-get upgrade

## Troubleshooted the Web Application to get it functioning correctly
1. `No such file or directory: client_secrets.json`
    * Apache Web Server has it's own working directory which the web application inherits, so needed to change _application.py_ to use the absolute path to _client_secrets.json_ instead of the relative path:
        `open('/var/www/itemcatalog/fsnd-item-catalog/client_secrets.json', 'r').read())['web']['client_id']`
        `oauth_flow = flow_from_clientsecrets('/var/www/itemcatalog/fsnd-item-catalog/client_secrets.json', scope='')`
    (https://code.google.com/archive/p/modwsgi/wikis/ApplicationIssues.wiki#Application_Working_Directory
    https://stackoverflow.com/questions/18454307/mod-wsgi-fails-when-it-is-asked-to-read-a-file)
2. `Google error 400. Error: origin_mismatch`
    * Needed to update the OAuth2 credentials to include the new web app URL.
        * Used the Google Cloud Platform Console to add the web app URL to the 
list of Authorised JavaScript origins and Authorised redirect URIs.
        * Downloaded the new JSON file and copied the contents to the _client_secrets.json_ file.
    (https://stackoverflow.com/questions/16850350/got-origin-mismatch-error-in-google-share-api)
3. `RuntimeError: the session is unavailable because no secret key was set`
    * Flask needs the _secret_key_ to be set before it allows a login_session to be used. When WSGI directs the /login request to the application, the _&lowbar;name&lowbar;_ parameter is not equal to _&lowbar;&lowbar;main&lowbar;&lowbar;_. Need to move the declaration of _secret_key_ to outside the `if __name__ == '__main__':` statement.
    (https://stackoverflow.com/questions/35657821/the-session-is-unavailable-because-no-secret-key-was-set-set-the-secret-key-on)