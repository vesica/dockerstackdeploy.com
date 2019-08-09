---
layout: post
author: Abd
---


Running your reverse proxy to serve domain names via HA Proxy on a Docker Swarm or Kubernetes with Let's Encrypt can be a challenge. When we set off to do this,
we had a few goals in mind:

* Auto-generate the certificates before they expire
* The ability to have virtually unlimited domain names served by the container
* Do not mount any volumes - so we can easily swap out the machines the containers are deployed on
* This should all be integrated into our CI/CD pipeline, so we can just change the HA Proxy config and push up to a git repository and the new config gets deployed

### The Solution

In the end, we came up with a simple solution, but it does require 2 private Docker images (to avoid having mounted volumes). Here's what we did:


1. Create a base container with HA Proxy, Let's Encrypt Certbot and all the required plugins that allow us to automatically generate certs.
2. Use this base container to build a private container which has the certificates inside it.
3. Use the second container and push your HA Proxy config file into it, deploy it to Docker Hub to Quay.io, and let a webhook deploy your updated configuration (this deployment bit is not covered in this article).

Because we use Certbot plugins and build the Docker images in Circle CI, our only limitation is that our domains' DNS be served by either Cloudflare, Linode, Route 53, or Google.


### Let's Docker Stack Deploy
#### Step 1: The base container with HA Proxy, Certbot and all the required Plugins

We've already done this, so you can simply pull the container from <a href="https://hub.docker.com/r/vesica/haproxy-certbot" target="_blank">vesica/haproxy-cerbot:latest</a>. You can also see the Dockerfile @ <a href="https://github.com/vesica/haproxy-certbot" target="_blank">https://github.com/vesica/haproxy-certbot</a>.


#### Step 2: Build the container that has all the certificates

Let's say you will be serving DNS for all your domains via Cloudflare. You will first need to create a .ini file with your Cloudflare API credentials, so Certbot can communicate with it.
Create a file called <code>cloudflare.ini</code> with the following:

<pre class="bg-light">
# cloudflare.ini

dns_cloudflare_email = your@cloudflare.account.email
dns_cloudflare_api_key = __API_KEY__
</pre>

Please do replace __API_KEY__ with your actual Cloudflare API key.


  Now let's create the <code>Dockerfile</code> for the container that will have the certificates:
  
<pre class="bg-light">
  # Dockerfile

  FROM vesica/haproxy-certbot:latest

  COPY cloudflare.ini /etc/cloudflare/cloudflare.ini

  RUN mkdir -p /etc/letsencrypt/merged-ssl/domain1.com/
  RUN touch /etc/letsencrypt/merged-ssl/domain1.com/domain1.com.pem

  ### CERTIFICATES TO ISSUE - ADD ANY NEW ONES AT THE BOTTOM

  RUN certbot certonly --dns-cloudflare --dns-cloudflare-credentials /etc/cloudflare/cloudflare.ini --dns-cloudflare-propagation-seconds 60 --non-interactive --agree-tos --email your@email.com \
  -d domain1.com \
  -d *.domain1.com \
  -d domain2.net \
  -d domain3.org \
  -d www.domain4.co.uk 

  ### DO NOT CHANGE ANYTHING ELSE ###

  COPY startup.sh /usr/local/bin/startup
  RUN chmod 755 /usr/local/bin/startup

  ENTRYPOINT ["startup"]

  CMD ["haproxy", "-f", "/usr/local/etc/haproxy/haproxy.cfg"]
</pre>

Let's go through the above Dockerfile and point our some of the finer points:

* We start by copying our cloudflare credentials file into the container.
* We create the directory and file where we will store the certificates for all the domains.
      <b>Note</b> that <code>domain1.com</code> should be replaced with whatever is the first domain in the list of domains you
      are generating certificates for.

* The rest is copying over a startup.sh file to start HA Proxy. We create this file below.

  Let's create the <code>startup.sh</code> file which will put the certificates in the right folder and start HA Proxy when the container runs.
  Add the following code to a file called <i>startup.sh</i>.

<pre class="bg-light">
# startup.sh

#!/bin/sh

## Put certs in usable place. Pay attention to domain1.com, that must always be the first domain. Otherwise change the below accordingly and haproxy.cfg too.
cat /etc/letsencrypt/live/domain1.com/fullchain.pem /etc/letsencrypt/live/domain1.com/privkey.pem > /etc/letsencrypt/merged-ssl/domain1.com/domain1.com.pem

# Start HA Proxy
set -e

# first arg is `-f` or `--some-option`
if [ "${1#-}" != "$1" ]; then
  set -- haproxy "$@"
fi

if [ "$1" = 'haproxy' ]; then
  shift # "haproxy"
  # if the user wants "haproxy", let's add a couple useful flags
  #   -W  -- "master-worker mode" (similar to the old "haproxy-systemd-wrapper"; allows for reload via "SIGUSR2")
  #   -db -- disables background mode
  set -- haproxy -W -db "$@"
fi

exec "$@"
</pre>


With the Dockerfile and startup.sh in place, let's build this container. Run <code>docker build . -t haproxy-certs:latest</code>.


#### Step 3: Build the container with the HA Proxy configuration

To build this container, we need to create a <code>Dockerfile</code> and an <code>haproxy.cfg</code> file.
Let's start with the Dockferfile.

<pre class="bg-light">
  # Dockerfile

  FROM haproxy-certs:latest

  COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg
</pre>


  Now let's create our <code>haproxy.cfg</code> file.

<pre class="bg-light">
  # haproxy.cfg

  #### MAKE CHANGES HERE ONLY IF YOU REALLY KNOW WHAT YOU ARE DOING #####
  #---------------------------------------------------------------------
  # Global settings
  #---------------------------------------------------------------------
  global
    daemon
    maxconn 10000
    tune.ssl.default-dh-param 2048

  #---------------------------------------------------------------------
  # common defaults that all the 'listen' and 'backend' sections will
  # use if not designated in their block
  #---------------------------------------------------------------------

  defaults

    mode                    http
    log                     global
    option                  dontlognull
    option                  http-server-close
    option                  redispatch
    retries                 3
    timeout http-request    1m
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 10000

  frontend haproxy-manager
    bind :::10000 v4v6
    mode http
    stats enable
    stats auth username:password
    stats refresh 15s
    stats show-node
    stats uri /haproxy-admin
    stats admin if TRUE

  frontend http-in
    bind :80 v4v6
    bind :::80 v6only
    bind :443 v4v6 ssl crt /etc/letsencrypt/merged-ssl/domain1.com/domain1.com.pem
    bind :::443 v6only ssl crt /etc/letsencrypt/merged-ssl/domain1.com/domain1.com.pem
    option forwardfor
    http-request set-header X-Forwarded-Proto https if { ssl_fc }
    http-request set-header HTTPS on if { ssl_fc }
    http-request set-header Ssl-Offloaded 1 if { ssl_fc }

  ## Define hosts
    acl host_domain1.com hdr(host) -i domain1.com

  ## Figure out which backend they will use
    use_backend domain1.com if host_domain1.com

  default_backend default

  backend default
    server w1 swarm-worker-1:port cookie S1 check
    server w2 swarm-worker-2:port cookie S1 check
    server w3 swarm-worker-3:port cookie S1 check

  backend domain1.com
    server w1 swarm-worker-1:port cookie S1 check
    server w2 swarm-worker-2:port cookie S1 check
    server w3 swarm-worker-3:port cookie S1 check
</pre>


  Let's build our container with the HA Proxy configuration and the certificates. Run <code>docker build . -t haproxy</code>.

  Once the container builds, you can push it up to Docker Hub or another container registry of your choice and deploy it.

  Once deployed, it will have a valid Let's Encrypt Certificate for all the domains.
