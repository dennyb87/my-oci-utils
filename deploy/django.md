# Deploy a Django app via Oracle Cloud Infrastracture  

```mermaid
flowchart LR
    subgraph oracle cloud instance
        nginx<-->gunicorn<-->django
    end
    client_x["client x"]<--"https"-->nginx
    client_y["client y"]<--"https"-->nginx
    client_z["client z"]<--"https"-->nginx
```

- [Deploy a Django app via Oracle Cloud Infrastracture](#deploy-a-django-app-via-oracle-cloud-infrastracture)
  - [Create Oracle Cloud instance](#create-oracle-cloud-instance)
  - [Create DuckDNS domain](#create-duckdns-domain)
  - [Open the ports](#open-the-ports)
    - [Ingress rules](#ingress-rules)
    - [Iptables](#iptables)
  - [Test with NGINX](#test-with-nginx)
  - [SSL certificate](#ssl-certificate)
  - [Configure Django](#configure-django)
  - [Gunicorn](#gunicorn)
  - [NGINX config](#nginx-config)


## Create Oracle Cloud instance  

Go to the [instances section](https://cloud.oracle.com/compute/instances) and create an instance.  

![oci_dashboard](https://github.com/dennyb87/elettrotecnica-serale/assets/7195133/06f79827-0c98-449a-940b-b998d75f0d2b)  
![oci_create_instance](https://github.com/dennyb87/elettrotecnica-serale/assets/7195133/c2e66b6f-6d67-4a91-867e-150b9b3729d6)  
![oci_image_shape](https://github.com/dennyb87/elettrotecnica-serale/assets/7195133/00aa4f24-2e81-43bb-916a-b3f44add215d)  
![oci_private_key](https://github.com/dennyb87/elettrotecnica-serale/assets/7195133/82dffafa-a35a-441b-8880-6da94aa10ef5)  

You can play with the configuration changing image or adding a block volume (persistent storage) incurring in zero costs as long as you're using **Always Free** services and resources. Finally remember to download the `private key` to access the instance via:  

```
chmod 600 private.key
ssh -i private.key <username>@<instance-ip-address>
```

The `ip address` of the instance can be found in the [instance details](https://cloud.oracle.com/compute/instances) as well as the `username`.  

> This guide assumes the image to be Ubuntu 22.04 where `username` is `ubuntu`  


## Create DuckDNS domain  

![duckdns](https://github.com/dennyb87/elettrotecnica-serale/assets/7195133/66490d29-3291-42b7-a225-a0075ca15de5)  

Oracle does not provide any public `FQDN`, I am sure there are many options here but I decided to create a [DuckDNS](https://www.duckdns.org/) account to get a free subdomain and configure it to point to my instance. It'll be something like:  

```
mylovelyapp.duckdns.org
```

## Open the ports  

At this point we need the instance to be able to communicate with the outside world, this is done in two steps:  

* **ingress rules** for VNIC
* **iptables** (for Ubuntu images)

### Ingress rules  

Ingress rules section can be reach from the [instance details](https://cloud.oracle.com/compute/instances) > subnet > security list > add ingress rules.  

![oci_subnet](https://github.com/dennyb87/elettrotecnica-serale/assets/7195133/6814e798-7913-45c6-8818-dca9157d3af1)  
![oci_security_list](https://github.com/dennyb87/elettrotecnica-serale/assets/7195133/1c3e6c8b-0f35-4d2c-97b4-a0651cf17816)  
![oci_ingress_rule](https://github.com/dennyb87/elettrotecnica-serale/assets/7195133/0f9dd184-6b50-4b8d-9c40-8ecfa912bbcc)  
![oci_add_ingress_rule](https://github.com/dennyb87/elettrotecnica-serale/assets/7195133/94245e3b-dae7-4032-ac06-26bd146549b2)  

Remember to add an ingress rule for both ports 80 and 443 respectively for http and https traffic.  

### Iptables  

Run the following to add the necessary netfilter rules.  

```
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT
sudo netfilter-persistent save
``` 

## Test with NGINX  

It's time to get some confidence by testing http communication.  

```
sudo apt update
sudo apt install nginx
```

After the installation the web server should be reachable at `http://mylovelyapp.duckdns.org` showing the nginx welcome page.  

![oci_nginx_welcome](https://github.com/dennyb87/elettrotecnica-serale/assets/7195133/ffd97f22-af49-4dae-a4ff-435bf669bf58)  


## SSL certificate  

Here we're going to create a [DV certificate](https://en.wikipedia.org/wiki/Domain-validated_certificate) using `certbot`, a client that fetches certificates from [Let's Encrypt](https://letsencrypt.org/).  

```
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx
```

The last command will fetch the certificate and amend the default nginx configuration to use it. The nginx welcome page will now be available also via https.  

## Configure Django  

It's time to start a Django project.  

```
sudo apt install python3-pip
pip install django
python3 -m django startproject mysite
```

Amend `settings.py` to configure `STATIC_ROOT` and `ALLOWED_HOSTS`

```
import os

...

STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
ALLOWED_HOSTS = ["mylovelyapp.duckdns.org"]
```

Collect the static files...  

```
cd ~/mysite
python3 manage.py collectstatic
```

## Gunicorn  

We're going to serve Django through [Gunicorn](https://gunicorn.org/), a python Web Server Gateway Interface (WSGI) HTTP server.  

```
sudo apt install gunicorn
```

Add the confguration file...  

```
sudo nano /etc/systemd/system/gunicorn.service
```
... with the following content:  

```
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/mysite/
ExecStart=/usr/bin/gunicorn --workers 3 --bind 127.0.0.1:8000 mysite.wsgi:application
```

We now want to reload the daemon to reread the service definition and restart the Gunicorn process.  

```
sudo systemctl daemon-reload
sudo systemctl restart gunicorn
```

## NGINX config  

All is left is to configure nginx to communicate with gunicorn, let's create a new nginx configuration for our lovely app.    

```
sudo nano /etc/nginx/sites-available/mylovelyapp
```

Here the nginx configuration.   

```
server {
    listen 80;
    server_name mylovelyapp.duckdns.org;
    
    location /static {
		alias /home/ubuntu/mysite/static;
	}

	location / {
		include proxy_params;
		proxy_pass "http://127.0.0.1:8000";
	}

    # managed by Certbot - start
	listen [::]:443 ssl ipv6only=on;
	listen 443 ssl;
	ssl_certificate /etc/letsencrypt/live/mylovelyapp.duckdns.org/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/mylovelyapp.duckdns.org/privkey.pem;
	include /etc/letsencrypt/options-ssl-nginx.conf;
	ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
    # managed by Certbot - end

	if ($scheme = http) {
		return 301 https://$server_name$request_uri;
	}
}
```

Note that the part `# managed by Certbot` can be copied from the default configuration which can now be unlinked.  

```
sudo unlink /etc/nginx/sites-enabled/default
sudo ln -s /etc/nginx/sites-available/mylovelyapp /etc/nginx/sites-enabled
```

Finally we need to add nginx user `www-data` to the group `ubuntu` to grant static files access.  

```
sudo gpasswd -a www-data ubuntu
```

After restarting nginx you should now see Django up and running!  

```
sudo systemctl restart nginx
```

![oci_django_running](https://github.com/dennyb87/elettrotecnica-serale/assets/7195133/14cb2291-b8c2-4aca-84bb-2cd2a2e38cc5)  
