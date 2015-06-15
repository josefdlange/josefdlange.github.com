---
title: "Setting Up a Private Docker Registry"
description: "Adventures in EC2 User-Data, Nginx Configuration, and that sweet, sweet Dockerized life."
layout: post
categories: [docker, ec2, aws, privacy, nginx]
---
Docker is a pretty cool technology. Abstracting much of the system configuration away from, well, system configuration, means that you can get to work writing and shipping code faster. Like most modern technologies, Docker is all-in on openness, which is great. Unless you're trying to run a business where your technology should remain private. 

The Docker Hub Registry offers a free private repository for any user, and more if they're willing to drop a couple bucks a month. Still, though, it's your code on someone else's servers. I trust GitHub to an extent but I'm cautious otherwise. In this abundance of caution, I started some research into how to get a truly private Docker Registry running.

The large part of doing so is actually really easy, since you can run an instance of the Docker registry just with a convenient "docker run", since the registry itself is an app container.

The following is an Amazon EC2 user-data script. If you start an instance with it, there will be a docker registry instance running, provided the instance's Instance Role allows read/write access to the S3 bucket you declare.

	#cloud-config
	runcmd:
	 - "echo 'Running Docker Setup and Registry'"
	 - "yum update -y"
	 - "yum install docker -y"
	 - "/etc/init.d/docker start"
	 - "docker run -d -e SETTINGS_FLAVOR=s3 -e AWS_BUCKET=[YOUR_S3_BUCKET] -e STORAGE_PATH=/registry -e STORAGE_REDIRECT=true -p '5000:5000' registry"
	 - "echo 'Done Setting up Docker'"

So far so good. Only problem is that, once you boot an instance with this, and throw it behind a Load Balancer (for simple SSL termination... you can configure SSL termination in the Docker Registry instance itself, but I'm lazy...) and point a DNS name at it, anyone can access it and push and pull images. So much for a private repository, right?

(Note -- I specifically haven't covered the SSL terminating load balancer and DNS stuff here. If you're not sure how to do all of that, please consult the Google)

Setting up auth for a Docker Registry is interesting, since it's actually not a feature of the registry itself. We have to go deeper!

Nginx is a great HTTP proxy and can have an auth layer built right in. Let's look into that. We'll want a pretty simple setup. First, install Nginx if not installed as well as http tools:

	sudo yum install nginx httpd-tools

Let's create a password file that Nginx will refer to for people trying to access your repo:

	sudo htpasswd -c /etc/nginx/docker-registry.htpasswd <USERNAME>  # Password asked by the script.

For adding more username/password entries, omit the "-c" option from the above command.

We'll want to configure Nginx by writing our own config to include to the master Nginx config. Put this in "/etc/nginx/sites-available/docker-registry":

	# For versions of Nginx > 1.3.9 that include chunked transfer encoding support
	# Replace with appropriate values where necessary

	upstream docker-registry {
	 server localhost:5000;
	}

	server {
	 listen 80;
	 server_name <YOUR DNS NAME>;

	 proxy_set_header Host       $http_host;   # required for Docker client sake
	 proxy_set_header X-Real-IP  $remote_addr; # pass on real client IP

	 client_max_body_size 0; # disable any limits to avoid HTTP 413 for large image uploads

	 # required to avoid HTTP 411: see Issue #1486 (https://github.com/dotcloud/docker/issues/1486)
	 chunked_transfer_encoding on;

	 location / {
	     # let Nginx know about our auth file
	     auth_basic              "Restricted";
	     auth_basic_user_file    docker-registry.htpasswd;

	     proxy_pass http://docker-registry;
	 }
	 location /_ping {
	     auth_basic off;
	     proxy_pass http://docker-registry;
	 }
	 location /v1/_ping {
	     auth_basic off;
	     proxy_pass http://docker-registry;
	 }

	}


Lastly, include it from your main Nginx config. It's located at `/etc/nginx/nginx.conf` Remove any `server` entries from the file and add this line inside the `http` section:

	include /etc/nginx/sites-available/*;

Mine, finished, looks like this:

	# For more information on configuration, see:
	#   * Official English Documentation: http://nginx.org/en/docs/
	#   * Official Russian Documentation: http://nginx.org/ru/docs/

	user  nginx;
	worker_processes  1;

	error_log  /var/log/nginx/error.log;
	#error_log  /var/log/nginx/error.log  notice;
	#error_log  /var/log/nginx/error.log  info;

	pid        /var/run/nginx.pid;


	events {
	    worker_connections  1024;
	}


	http {
	    include       /etc/nginx/mime.types;
	    default_type  application/octet-stream;

	    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
	                      '$status $body_bytes_sent "$http_referer" '
	                      '"$http_user_agent" "$http_x_forwarded_for"';

	    access_log  /var/log/nginx/access.log  main;

	    sendfile        on;
	    #tcp_nopush     on;

	    #keepalive_timeout  0;
	    keepalive_timeout  65;

	    #gzip  on;

	    # Load modular configuration files from the /etc/nginx/conf.d directory.
	    # See http://nginx.org/en/docs/ngx_core_module.html#include
	    # for more information.
	    include /etc/nginx/conf.d/*.conf;

	    include /etc/nginx/sites-available/*;   ## ADDED THIS

	}

If everything goes as planned, you should now be able to send the following:

	curl localhost/_ping

It should return just `{}`. 

Cool. Now we have successfully spun up a server and manually set up its nginx proxy to enable auth. How do we glue this all together? I'd recommend throwing the above into the user-data. It grabs the relevant config files from S3 and puts them in place where they belong. Here's a good example:

	#!/bin/bash
	echo 'Running Docker Setup and Registry'
	yum update -y
	yum install docker httpd-tools nginx -y
	/etc/init.d/docker start
	docker run -d -e SETTINGS_FLAVOR=s3 -e AWS_BUCKET=[YOUR BUCKET HERE] -e STORAGE_PATH=/registry -e STORAGE_REDIRECT=true -p '5000:5000' registry
	echo 'Done Setting up Docker'
	aws s3 cp s3://[YOUR_BUCKET_HERE]/config/nginx.conf /etc/nginx/nginx.conf
	mkdir -p /etc/nginx/sites-available
	aws s3 cp s3://[YOUR_BUCKET_HERE]/config/docker-registry /etc/nginx/sites-available/docker-registry
	# At your discretion: either store your htpasswd file in S3 with everything else, or set passwords on boot:
	# aws s3 cp s3://[YOUR_BUCKET_HERE]/config/docker-registry.htpasswd /etc/nginx/docker-registry.htpasswd
	# --- OR ---
	# htpasswd -c -b /etc/nginx/docker-registry.htpasswd [USERNAME1] [PASSWORD1]   # -c flag for creating file.
	# htpasswd -b /etc/nginx/docker-registry.htpasswd [USERNAME2] [PASSWORD2]
    /etc/init.d/nginx start
	

Any problems? Feel free to ask a question in the comments -- I wrote this _as_ I was figuring it out, so it's likely I've missed something here and there.

Inspired and compiled with these posts in mind:

[Build Your Own Private Docker Registry for $3/month](http://blog.50projects.com/2014/08/build-your-own-private-docker-registry.html)
[How To Set Up a Private Docker Registry on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-private-docker-registry-on-ubuntu-14-04)
