Complete guide to install && configure Hopper.pw on Debian (and Ubuntu) server by j0eblack ( [@RedSunEmpire](https://github.com/RedSunEmpire) ).

First things first, clone the repo to your working direcroy.

Since we are going to set-up Hopper to run as non-privileged user, you should clone it in the home directory of a unix user:

```bash
root@deb: adduser hopper
root@deb: su hopper
hopper@deb: cd ~
hopper@deb: git clone https://github.com/asmaps/hopper.pw .

# Lets isntall a lot of essential software packages:

root@deb: apt-get install postgresql postgresql-contrib libpq-dev python-dev nginx git python-pip bind9 bind9utils libmemcached-dev

# Install all of the required Python modules:

root@deb: pip install -r requrements.txt
root@deb: pip install supervisor gunicorn 

# Create the Postgresql user and database:

root@deb: su postgres
postgres@deb: createuser
# Enter name of role to add: hopper
# Shall the new role be a superuser? (y/n) n
# Shall the new role be allowed to create databases? (y/n) n
# Shall the new role be allowed to create more new roles? (y/n) n

postgres@deb: createdb hopper

# Grant the user access to the database:

postgres@deb: psql
postgres=# alter user hopper with encrypted password 'somepass';
postgres=# grant all privileges on database hopper to hopper;
postgres=#\q

# Lets create our local_settings file that will be used by Hopper.

hopper@deb: cd
hopper@deb: cd hopperpw/hopperpw/settings/
hopper@deb: cp local_settings.py.example local_settings.py
hopper@deb: vim local_settings.py
```

Delete all the content of the file and paste the following replacing the database name, username and password with yours:

```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'hopper',
        'USER': 'hopper',
        'HOST': 'localhost',
        'PASSWORD': 'somepass',
    }
}

SECRET_KEY = 'some_secret_key'
SESSION_COOKIE_SECURE = False
INTERNAL_IPS = ['192.168.1.28'] # replace it with the IP address wiht yours
BASEDOMAIN = 'boobs.com' # replace with your domain name
#DEFAULT_FROM_EMAIL = 'acc@gmail.com'
#EMAIL_HOST = 'smtp.gmail.com'
#EMAIL_PORT = 587
#EMAIL_USE_TLS = True
#EMAIL_HOST_USER = 'acc@gmail.com'
#EMAIL_HOST_PASSWORD = 'pass'
DEBUG = False
```

As you can see there are commented lines which you can change and use Gmail to send account activation emails to users who register on your website.

After saving the file, you can start the Django application by running the following two commands in the directory `/home/hopper/hopperpw/hopperpw`:

```
python manage.py migrate
python manage.py runserver
```

At this point you should not receive any error message and you can visit the default Hopper web-page on the URL provided from the ouput.

However, static resources are not being loaded and the pages will load half-broken.

To serve the static files over Nginx for better perfomance, we will need to edit the Nginx config file later on.

The first thing you want to do it is create a admin user with the following command:

```
hopper@deb: python manage.py createsuperuser
```

You will be prompted to set a email address and password for the new admin user.

In my set-up I chose to use Bind for managing the DNS zones and that is what I will describe in this guide.

You can also use PowerDNS as the creator of the project did.

We have installed bind and now we will need to create the directory in which the zones will be stored:

```
root@deb: cd /etc/bind/
root@deb: mkdir -p zones/master
```

Now we will need to create the zone file that will have all of the DNS records for the domain name.

Use your favourite text editor to create the file `/etc/bin/zones/master/db.domain.com` and add the following content to it:

```
$TTL    3h  
@       IN      SOA     ns1.domain.com. admin.domain.com. (
                          1        ; Serial
                          3h       ; Refresh after 3 hours
                          1h       ; Retry after 1 hour
                          1w       ; Expire after 1 week
                          1h )     ; Negative caching TTL of 1 day 
;
@       IN      NS      ns1.domain.com.


domain.com.    IN      MX      10      mail.domain.com.; add this if you have a mail server
domain.com.    IN      A       192.168.1.28; the IP address of your machine
ns1                     IN      A       192.168.1.28
ns2                     IN      A       192.168.1.28
ipv4.www                IN      A       192.168.1.28
```

You will need to replace domain.com with your own domain name and the IP address of the server where you are setting Hopper to run.

To be able to update the A records successfully, we will need to generate a TSIG key for our domain name using the following command:

```
root@deb: cd /etc/bind/
root@deb: dnssec-keygen -r /dev/urandom -a HMAC-MD5 -b 512 -n HOST domain.com
```

Now you will have the following files:

* Kdomain.com.+157+18137.key
* Kdomain.com.+157+18137.private

The information that you will need is located in the .private file.

The domain name,zone file and private key need to be added to the main Bind config `/etc/bind/named.conf.local` and add the following to it:

```
zone "domain.com" {
     type master;
     file "/etc/bind/zones/master/db.domain.com";
     allow-update { key "domain.com."; };
     journal "/var/lib/bind/zone.domain.com";
     };
key "domain.com." {
    algorithm hmac-md5;
    secret
"thekeystring==";
};
```

The key string can be found in the .private file.

Now it is time to restart bind and check it any errors will be logged in `/var/log/daemon.log`

Just to be safe, you can change the ownership of the directory `/var/lib/bind` to bind:bind and make it with permissions 770 so the bind serivece can properly write in it.

To check if the nsupdate command is working, we can create a file nsupdate.txt with the following info in it:

```
server domain.com
debug yes
zone domain.com.
update add testnsupdate.domain.com. 86400 A 192.168.1.28
show
send
```

After you same the file, you should run this command:

```
root@deb: nsupdate -k Kdomain.com.+157+18137.private -v nsupdate.txt
```

If no error messages are retuned, the following lines should be present in log `/var/log/daemon.log`:

Mar  5 14:51:18 deb named[3683]: client 192.168.1.28#55743: signer "domain.com" approved
Mar  5 14:51:18 deb named[3683]: client 192.168.1.28#55743: updating zone 'domain.com/IN': adding an RR at 'testnsupdate.domain.com' A

Another way to check if everyting is working is by lookin up the test sub-domain on the server:

```
root@deb: dig testnsupdate.domain.com @192.168.1.28

;; QUESTION SECTION:
;testnsupdate.domain.com.		IN	A

;; ANSWER SECTION:
testnsupdate.domain.com.	86400	IN	A	192.168.1.28

;; AUTHORITY SECTION:
domain.com.		10800	IN	NS	ns1.domain.com.

;; ADDITIONAL SECTION:
ns1.domain.com.		10800	IN	A	192.168.1.28
```

Great, nsupdate is working properly.

Now lets get back to setting Hopper with Nginx and Gunicorn.

At the start of the guide gunicorn was installed and we can test if it can start the Hopper project by running the following command:

```
root@deb: cd /home/hopper/hopperpw/hopperpw
root@deb: gunicorn hopperpw.wsgi:application --bind 192.168.1.28:8001 --env DJANGO_SETTINGS_MODULE='hopperpw.settings.dev'
```

Now if you vist the IP address on port 8001 you will see the application working but again without the static files.

To be easier for us to run the application, we will create a bash script (run_hopper.sh in my case) with the following content:

```
#!/bin/bash

NAME="hopperpw"                                  # Name of the application
DJANGODIR=/home/hopper/hopperpw/hopperpw             # Django project directory
SOCKFILE=/home/hopper/hopperpw/hopperpw/run/gunicorn.sock  # we will communicte using this unix socket
USER=hopper                                        # the user to run as
GROUP=hopper                                     # the group to run as
NUM_WORKERS=3                                     # how many worker processes should Gunicorn spawn
DJANGO_SETTINGS_MODULE=hopperpw.settings.dev             # which settings file should Django use
DJANGO_WSGI_MODULE=hopperpw.wsgi                     # WSGI module name

echo "Starting $NAME as `whoami`"

# Activate the virtual environment
cd $DJANGODIR
export DJANGO_SETTINGS_MODULE=$DJANGO_SETTINGS_MODULE
export PYTHONPATH=$DJANGODIR:$PYTHONPATH

# Create the run directory if it doesn't exist
RUNDIR=$(dirname $SOCKFILE)
test -d $RUNDIR || mkdir -p $RUNDIR

# Start your Django Unicorn
# Programs meant to be run under supervisor should not daemonize themselves (do not use --daemon)
exec /usr/local/bin/gunicorn ${DJANGO_WSGI_MODULE}:application \
  --name $NAME \
  --workers $NUM_WORKERS \
  --user=$USER --group=$GROUP \
  --bind=unix:$SOCKFILE \
  --log-level=info \
  --log-file=/home/hopper/hopperpw/hopperpw/err.log \
  --access-logfile=/home/hopper/hopperpw/hopperpw/access.log
```

Error logs and access logs will be saved in our hopperpw directory.

From now on we can use this bash script in order to start the application.

Lets configure Nginx to load the static files for us.

We will edit the config file `/etc/nginx/sites-enabled/default` and replace all of the content in it with the following:

```
upstream hopperpw {
	server unix:/home/hopper/hopperpw/hopperpw/run/gunicorn.sock fail_timeout=0;
}
server {
	listen 80;
	server_name domain.com;
	client_max_body_size 4G;
	location /static/ {
		alias /home/hopper/hopperpw/hopperpw/static/;
	}
	location / {
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header Host $host;
	port_in_redirect off;
	if (!-f $request_filename) {
            proxy_pass http://hopperpw;
            break;
        }
	}
}
```

You will need to alter the paths to match your Hopperpw project.

After you save the file and restart Nginx, you should also start the Hopper project using the bash script that we created:

```
hopper@deb: bash run_hopper.sh &
```

Now if you open your domain name, the pages will start loading perfectly.

The next step is to log in with the admin user and add the domain name and host in the backend of the application.

You can use the URL: domain.com/admin to log in and after that you will need to Click on > domains > add domain.

There you will need to use the key from `/etc/bind/Kdomain.com.+157+18137.private` file and set the IP address of your server.

After you save it, you will need to click on Sites > click on 'example.com' and change the Domaina name and Display name to your own domain name.

Once that is saved, you can navite back to the website and click on Overview.

There you can create a test sub-domain 'test', select 'domain.com' from the drop-down and hit Create.

You should be greeted with a successful creation of the sub-domain.

Keep in mind that you will need to change the default 'hopper.pw' domain name to your own in some .html files.

I am not goind to explain that as you will need to grep recursivery in the directory of hopper and replace all of the occurences of the domain name.

To test if the remote A record update works, you can copy the IPv4-Update URL and make a curl request to it from a differet PC.

I have also used the Supervisor tool in order to start the Hopper project using the gunicord bash script that we created earlier.

You can create the following `hopper.conf` file in `/etc/supervisor/conf.d/` directory:

```
[program:hopper]
command = /home/hopper/hopperpw/hopperpw/run_hopper.sh
user = hopper
stdout_logfile = /home/hopper/hopperpw/hopperpw/supervisor.log
redirect_stderr = true
environment=LANG=en_US.UTF-8,LC_ALL=en_US.UTF-8
```

Create the `supervisor.log` with:

```
hopper@deb: touch /home/hopper/hopperpw/hopperpw/supervisor.log
```

Run these two commands in order to add the new config file:

```
root@deb: supervisorctl reread
root@deb: supervisorctl update
```

If you encounter any errors while you are doing the setup, you should use Google in order to resolve them.

Here are all of the articles that I used in the setup process:

https://linuxconfig.org/linux-dns-server-bind-configuration
http://agiletesting.blogspot.bg/2012/03/dynamic-dns-updates-with-nsupdate-and.html
http://michal.karzynski.pl/blog/2013/06/09/django-nginx-gunicorn-virtualenv-supervisor/

I hope this guide will help some people in setting up their own dynamic DNS server.
