# nginx-server-blocks
How to set up a single server for use with multiple Django web applications via Nginx server blocks.

## Notes

For security purposes, each webapp should be run by a new, independent user on the system. Each user should have a $HOME pointing to `/srv/<application>`, which hosts the directory and config files. Each application should also then be run inside a separate virtualenv.

Information from:
- http://michal.karzynski.pl/blog/2013/06/09/django-nginx-gunicorn-virtualenv-supervisor/
- http://nginx.org/en/docs/http/server_names.html
- http://supervisord.org/running.html

## Requirements
#### A non-root user with `sudo` priveliges.
- virtualenv (whilst it is usually bad practice to use `sudo pip`, this ensures virtualenv is compatible with both Python versions: 
```bash
$ sudo yum install python-virtualenv
$ sudo pip install virtualenv
```
#### supervisor
- Install and set permissions for accessing socket files
```bash
$ sudo yum install supervisor

- Edit the config file to (a) provide access to socket files, and (b) set up individual program files.
```bash
$ sudo vim /etc/supervisord.conf

# alter [unix_http_server]:
;chmod=0766
;chown=root:supervisor

# alter [include]:
files = supervisord.d/*.ini supervisord.d/*.conf
```
#### Nginx: 
```bash
$ sudo yum install nginx
```


## Environment set-up

1. Create a system group called `webapps`. All system users which run webapps will be members of this group.

```bash
$ sudo groupadd --system webapps
```

2. Create a system group called `supervisor`. The sudo user will be part of this, and it gives access to the socket files for supervisor.

```bash
$ sudo groupadd supervisor
$ sudo usermod -a -G supervisor `whoami`
```

3. Start Nginx
```bash

sudo firewall-cmd --permanent --zone=public --add-port=80/tcp
sudo firewall-cmd --reload

sudo service nginx start
```
These commands should be re-run for every port you want to use.

Test that  Nginx has initialised correctly and is running by navigation to the IP of the server. It should display an Nginx welcome page. Additionally, ensure that the Nginx config is set up for adding applications:

```bash
# if you do not have the following directory, make it:
$ sudo mkdir /etc/nginx/sites-available

# additionally, check that one of the following lines is present within /etc/nginx/nginx.conf:
include /etc/nginx/conf.d/*.conf;
# -- OR --
include /etc/nginx/sites-enabled/*.conf;

# whichever of these is being included, use this for the symlink when adding new applications (will do this in the last step of this procedure). Enabled application config files should symlink to here.
```

NOTE: all group changes require a new login to set groups.

## Adding a new application

1. Create a new user for the application with a home directory where the application code will be hosted.

```bash
$ sudo useradd --system --gid webapps --shell /bin/bash --home /srv/<application_name> <user_name>
$ sudo mkdir -p /srv/<application_name>
$ sudo chown <user_name> /srv/<application_name>
```

2. Set up a virtual environment for the application. 

```bash
$ su - <user_name>
$ virtualenv --python=/usr/bin/python[2.7|3.6] .
$ source bin/activate  # to enter the venv

# If the project has a requirements.txt file, run:
(<application_name>) $ pip install -r requirements.txt

# If not, pip install each required module into the virtualenv.
```

3. Add the Django project to the $HOME directory (e.g. through git clone).

4. Install gunicorn to the virtual environment.
```bash
(<application_name>) $ pip install gunicorn
```

5. Create a script to start gunicorn so it will serve the dynamic content of the app, then set the script as executable.
```bash
(<application_name>) $ touch bin/gunicorn_start
(<application_name>) $ deactivate
$ logout
$ sudo chmod u+x /srv/<application_name>/bin/gunicorn_start
```

6. Write the following to `gunicorn_start`:
```bash
#!/bin/bash

NAME="<application_name>"
VIRTUALENV=/srv/<application_name>/bin/activate
GUNICORN=/srv/<application_name>/bin/gunicorn
DJANGODIR=/srv/<application_name>/<django_project_directory>
SOCKFILE=/srv/<application_name>/run/gunicorn.sock
USER=<user_name>
GROUP=webapps
NUM_WORKERs=3  # should be 2 * # CPUs + 1
# NB: <django_project_name> differs from <application_name>; it is set when running 'django-admin startproject'
DJANGO_SETTINGS_MODULE=<django_project_name>.settings  # by default; this is project-dependent
DJANGO_WSGI_MODULE=<django_project_name>.wsgi


echo "Starting $NAME as `whoami`."

# activate virtualenv
cd $DJANGODIR
source $VIRTUALENV
export DJANGO_SETTINGS_MODULE=$DJANGO_SETTINGS_MODULE
export PYTHONPATH=$DJANGODIR:$PYTHONPATH

# create run dir if it doesn't exist
RUNDIR=$(dirname $SOCKFILE)
test -d $RUNDIR || mkdir -p $RUNDIR

# start Django Unicorn
exec $GUNICORN ${DJANGO_WSGI_MODULE}:application \  
  --name $NAME \
  --workers $NUM_WORKERS \
  --user=$USER --group=$GROUP \
  --bind=unix:$SOCKFILE \
  --log-level=debug \
  --log-file=-
```

Check that the script can execute without error by running `bin/gunicorn_start`.

7. Create a program config file to allow supervisor to manage starting, automatic rebooting on failure, and starting on system start. Expected outcome is the running of the webapp's server by gunicorn.

```bash
$ sudo vim /etc/supervisord.d/<application_name>.conf

# add the following:
[program:<application_name>]
command = /srv/<application_name>/bin/gunicorn_start                    ; Command to start app
user = <user_name>                                                      ; User to run as
stdout_logfile = /srv/<application_name>/logs/gunicorn_supervisor.log   ; Where to write log messages
redirect_stderr = true                                                  ; Save stderr in the same log

# now create the log files
$ sudo su - <user_name>
$ mkdir -p /srv/<application_name>/logs
$ touch /srv/<application_name>/logs/gunicorn_supervisor.log
```

Check that the program is available by running (as the user with sudo priveliges) `sudo supervisorctl reread`. Expected stdout is `<application_name>: available`. Once this is confirmed, start the app using supervisor:

```bash
$ sudo supervisorctl update
<application_name>: added process group
```

The application can be monitored and started/stopped with the following commands:
```bash
$ sudo supervisorctl status <application_name>                       
<application_name>                            RUNNING    pid 18020, uptime 0:00:50
$ sudo supervisorctl stop <application_name>  
<application_name>: stopped
$ sudo supervisorctl start <application_name>                        
<application_name>: started
$ sudo supervisorctl restart <application_name> 
<application_name>: stopped
<application_name>: started
```

8. Set up the Nginx virtual server configuration for the application. The following file should be created at `/etc/nginx/sites-available/<application_name>.conf`:

```bash
upstream <application_name>_server {

  server unix:/srv/<application_name>/run/gunicorn.sock fail_timeout=0;
}

server {

    # adding default_server means it will be the first to respond to a specific port request.
    # if no port is specified in the request, 80 is the deafult HTTP port (in nginx.conf)
    listen   <port> default_server;
    server_name <host_ip>;

    client_max_body_size 4G;

    access_log /srv/<application_name>/logs/nginx-access.log;
    error_log /srv/<application_name>/logs/nginx-error.log;
 
    location /static/ {
        alias   <django_app_static_dir>;
    }
    
    location /media/ {
        alias   <django_app_media_dir>;
    }

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # enable this if and only if you use HTTPS, this helps Rack
        # set the proper protocol for doing redirects:
        # proxy_set_header X-Forwarded-Proto https;

        proxy_set_header Host $http_host;
        proxy_redirect off;

        # Try to serve static files from nginx
        if (!-f $request_filename) {
            proxy_pass http://<application_name>_server;
            break;
        }
    }

    # Error pages if you have custom ones set up
    error_page 404 /404.html;
    location = /404.html {
        root <django_static_dir>;
    }
    
    error_page 500 502 503 504 /500.html;
    location = /500.html {
        root <django_static_dir>;
    }
}
```
If the Django project does not have static/media directories, the corresponding sections can be removed.

Once this file is written, symlink it into the active config directory for Nginx defined in `nginx.conf` (either /etc/nginx/conf.d/ or /etc/nginx/sites-enabled/):

```bash
$ sudo ln -s /etc/nginx/sites-available/<application_name>.conf <nginx_conf_dir>/<application_name>.conf
$ sudo nginx -s reload
```

With gunicorn and nginx running, the webapp should now be accessible at the IP and port specified. 

Troubleshooting: check log files (<application_name>/logs/), check file permissions and double check files created in this procedure.
