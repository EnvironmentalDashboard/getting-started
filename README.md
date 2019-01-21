# getting-started
Information about the general architecture of our server, our workflow, and other DevOps related information needed to get started working effectively at Environmental Dashboard.

## Table of Contents
1. [General information](#general-information)
2. [Style guide](#style-guide)
3. [Server file structure](#server-file-structure)
4. [Local development](#local-development)
5. [DNS and Load Balancing](#dns-and-load-balancing)
6. [SSL](#ssl)

## General information
Environmental Dashboard is comprised of several web apps
- Citywide Dashboard
	- Animated display of resource use and environmental conditions
- Building Dashboard
	- d3.js-powered time series and Lucid-powered pages
- Community Voices
	- MVC-engineered blogging and digital signage app
- Events Calendar
	- Simple flat PHP calendar
- Environmental Orb
	- Node.js powered orb server, at environmentalorb.org

This content is served from two servers, 159.89.232.129 (DigitalOcean server, referred to as nyc1 in haproxy configs) and 132.162.36.210 ("CSR" server, referred to as node in haproxy configs). The 159 address is the primary IP and the DigitalOcean server is used as both a web server and a load balancer, offloading some traffic to the CSR server (see [load balancing](#dns-and-load-balancing) for more information). Thus, instances of each web app need to be maintained on each server and static assets syned (which is not yet implemented).

## Style guide
php-cs-fixer

## Server file structure
Important locations:

- `/var/www` www resources (i.e. web apps)
- `/var/www/repos` contains all repos for www apps (e.g. environmentaldashboard.org, community-voices)
- `/var/www/uploads` all apps place user uploaded content here
- `/var/repos` contains all repos for non-www apps (e.g. relative-value, scripts)
- `/var/secret` sensitive information e.g. db passwords

Symlinks are used extensively to create a more flexible file structure e.g. the web app `GoogleDriveAPI` can be symlinked into the website (`ln -s /var/www/repos/GoogleDriveAPI /var/www/environmentaldashboard.org/google-drive`) so its URL can be customized and independant of the repo name.


## Local development
Docker is important to understand as it is used to run some apps (e.g. the data collection daemons and community voices) in production and can be used to easily set up a development environment (though it's possible to use other tools such as MAMP). First, `cd` into your development environment (we will assume that is `~/repos`), download, and run the following Dockerfile:
```
FROM ubuntu:18.04
ENV DEBIAN_FRONTEND=noninteractive
ENV APACHE_RUN_USER=www-data APACHE_RUN_GROUP=www-data APACHE_LOG_DIR=/var/log/apache2 APACHE_LOCK_DIR=/var/lock/apache2 APACHE_PID_FILE=/var/run/apache2.pid
ENV TZ=America/New_York
# timezone: https://serverfault.com/a/683651/456938
RUN apt-get update && \
  apt-get -qq -y install apt-utils tzdata apache2 php libapache2-mod-php php-mysql curl && \
  ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone && \
	a2enmod access_compat alias auth_basic authn_core authn_file authz_core authz_host authz_user autoindex deflate dir env filter headers mime mpm_prefork negotiation php7.2 proxy_http proxy reqtimeout rewrite setenvif socache_shmcb ssl status
EXPOSE 80
CMD /usr/sbin/apache2ctl -D FOREGROUND
# to run:
# docker build -t dev-server .
# docker run -dit -p 80:80 -v $(pwd):/var/www/html/repos --name dev-server1 dev-server
```
Cloning repositories into `~/repos` will make them available at `http://localhost/name-of-repo`. However, the orb-server and community voices apps have special Dockerfiles that need to be used to properly install them.


## DNS and load balancing
When someone visits environmentaldashboard.org:
1. DNS A records resolve to 159.89.232.129 IP address, user requests page from this IP (if you visit http://159.89.232.129:80 in your browser however it will not give you the website as apache has a virtual host configuration which looks at the Host: header i.e. the domain name to determine the correct config block to use.)
2. If a user requests a page (from the 159 IP but using the domain name) over port 80, apache responds with a redirect to port 443 where HAProxy is listening. if you visit https://159.89.232.129:443 in your browser (ignore the ssl warning), this will hit up HAProxy directly without DNS resolution and perform the next step (3)
3. HAProxy will choose to offload your request to either the DigitalOcean server (localhost in this case, the 159 address) or the CSR server (132.162.36.210). because HAProxy is running on 443, the DigitalOcean server handles ssl over 444. So, you’ll end up being served by either https://159.89.232.129:444 or https://132.162.36.210:443 which you should be able to pull up directly. Both these machines in addition to having apache running on 444/443, have community voices set up at port 3002/5297: http://159.89.232.129:3002 and http://132.162.36.210:5297.

## SSL
Our SSL certificates are issued by Lets Encrypt and conveniently renewed by `certbot` on nyc1 and then distributed to nodes by [`scripts/sync_ssl.sh`](https://github.com/EnvironmentalDashboard/scripts), which is called by the certbot [webhook](https://github.com/EnvironmentalDashboard/et-cetera/blob/master/letsencrypt/cli.ini). See the [documentation](https://certbot.eff.org/docs/using.html#re-creating-and-updating-existing-certificates) for managing these certificates. `certbot certificates` will print a list of certificates. Updating existing certificates can be accomplished with the `--expand` option e.g. `certbot --expand -d environmentaldashboard.org -d www.environmentaldashboard.org -d api.environmentaldashboard.org -d phpmyadmin.environmentaldashboard.org`.
