---
title: 'Nginx: Reverse Proxy'
date: '24-09-2017 22:00'
categories:
  - 'Blog'
tag:
  - 'Nginx'
---

In this article we will look at what a reverse proxy is, as well as how to set one up on CentOS using Nginx.

## What is a Reverse Proxy

A reverse proxy is a type of proxy server which retrieves resources on behalf of a client, from one or more servers. The collected information is then returend to the client as if it originated from the web server itself. It is the opposite of a forward proxy (one which allows it's clients to contact any server), a reverse proxy allows it's servers to be contacted by any client.

## Why do I need a Reverse Proxy

You would need a reverse proxy for a number of potential use cases. The most common reason to use a reverse proxy is to increase security. For example, a reverse proxy can be a hardend computer in your DMZ, which then takes client traffic from the internet, and passes it to your internal web application servers. This would have two main security benefits:

1. Lower attack surface: Each system you expose to the internet increases the attack surface to get into your network. It only takes the compromise of one of those systems to wreak havoc on all of your internal systems.

2. Easier to patch/manage/secure: Having less resources exposed to the internet also means that you have less critial infrastructure to patch, manage, and secure. Of course, you should still definitely be doing patches on your internal systems, however, the best security is defence in depth, basically, having as many hard layers to go through before the attack is able to breach your inner network. Having a reverse proxy increases the number of layers before getting into your network.

Another reason you would want to setup a reverse proxy is if you have a number of resources inside your network, such as web and application servers/services, which you want to all be accessable outside of your network, but you only have one public IP address. Instead of having websites on different services, you can put all of those sites behind your reverse proxy on one IP and port (80/443), and then have clients connect to the proper backend based on the (sub)domain on which they are trying to connect.

Reverse Proxies also decrease your management costs, for example, instead of getting an SSL certificate and installing it on every web server you have, and then managing all of those certificates, you can get one certificate (either wildcard or SAN), and install it on your reverse proxy. This way, all traffic between the client and web server is encryted, but you don't have to spend extra time and money dealing with installing certificates on each service behind the proxy. Note however, that doing this means that traffic between the reverse proxy and web server (inside your network) is unencrypted, unless you configure the internal site for HTTPS. This is usually not a big deal for a homelab environment or testing, but please be advised that it is less secure and should not be used in production environment. If you are in a production environment, ensure that traffic is encrypted between the reverse proxy and the internal application server.

## What Reverse Proxy should I use

There are many types of reverse proxies, from dedicated enterprise hardware/software like F5 Big-IPs, all the way down to a regular web server, like Nginx or Apache. Each of these will have different Pros and Cons, which you should research yourself. However, for a small blog or homelab environment, a reverse proxy running Nginx will be fine.

## Installing Nginx as a Reverse Proxy

Now that we have looked at what a Reverse Proxy is, and why we might want to use one, lets look at installing Nginx as a reverse proxy.

### Step 1: Installing Nginx

In this step, we will look at installing Nginx on CentOS 7. Note that these steps should be similar for your prefered distribution, however, I make no promises.

First, update packages on your CentOS server:

```sh
$ sudo yum -y update
```

Install Nginx:

```sh
$ sudo yum -y install nginx
```

Start Nginx and enable (so that it starts at boot:

```sh
$ sudo systemctl start nginx
$ sudo systemctl enable nginx
```

Check that Nginx is running:

```sh
$ sudo systemctl status nginx
```

Once Nginx is running, we need to open the firewall so that our clients can access it.

```sh
$ sudo firewall-cmd --permanent --add-service=http
$ sudo firewall-cmd --permanent --add-service=https
$ sudo firewall-cmd --reload
```

Now that we have done that, we should verify that we are actually able to reach nginx. Go to `http://<ip_of_your_host>` in a web browser, and you should get an Nginx page. If you do not, ensure that Nginx is running, and that you have opened the firewall.

### Step 2: Configuring Nginx

In this step, we will configure some sane defaults for our Nginx installation, including SSL and Proxy defaults.

First, let's look at using Nginx's snippets feature to configure the default SSL and Proxy settings in Nginx.

#### Proxy Defaults

In `/etc/nginx/snippets/`, create a file called `proxy-defaults.conf`, with the contents of:

```nginx
proxy_redirect off;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $remote_addr;
```

Then add `include snippets/proxy-defaults.conf` to your server block.

#### SSL Defaults

First, let's take a look at setting up the SSL certificates and dhparam files needed.

##### Getting an SSL certificate

We will look at setting up Let's Encrypt, a free certificate authority.

###### Step 1 - Installing the Certbot Let's Encrypt Client

Enable the EPEL repository:

```sh
$ sudo yum -y install epel-release
```

Install certbot-nginx:

```sh
$ sudo yum -y install certbot-nginx
```

For Cerbot to work, you need to specify your server name in `/etc/nginx/nginx.conf`

```sh
$ sudo vi /etc/nginx/nginx.conf
```

Find the existing `server_name` line:

```nginx
server_name _;
```

Replace the `_` underscore with your domain name:

```nginx
server_name example.com www.example.com;
```

Verify that the syntax of the configuration file is correct:

```sh
$ sudo nginx -t
```

If this completes without errors, reload Nginx to load the new configuration:

```sh
$ sudo systemctl reload nginx
```

###### Step 2 - Obtaining a Certificate

There are a number of ways to obtain certificates from Let's Encrypt through Certbot. We will use the Nginx plugin to take care of reconfiguring Nginx and reloading the config when certificates are updated.

```sh
$ sudo certbot --nginx --rsa-key-size 4096 -d example.com -d www.example.com
```

This will run `certbot` with the `--nginx` plugin, a key size of 4096 (more secure than the default 2048), and use `-d` to specify the domain names we would like the certificate to be valid for.

! You need to add every sub-domain you want to use Let's Encrypt certificates, seperated with the `-d` flag.

If this is the first time you've run certbot on this server, you will be prompted to enter an email address and agree to the terms of service. It will then test to ensure that it can communicate with Let's Encrypt's servers.

Once that is complete, you will be asked how you'd like to configure HTTPS on your server. Choose the option you'd like to use, and then press Enter. You will then be told where your certificates are stored on the server.

###### Step 3 - Updating Diffie-Hellman Parameters

Currently, if you test your server using the [SSL Labs test](https://www.ssllabs.com/ssltest/), you will notice that you only get a **B** grade. We can fix this be creating a new dhparam.pem file and adding it to our `ssl-defaults.conf` file (see the next section).

```sh
$ sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 4096
```

This will take a while. Make note of where the dhparam.pem file is located, as it will be needed in the next step.

###### Step 4 - Setting Up Auto Renewal

Let's Encrypts certificates are only valid for ninety days. Therefore, we will need to setup a command to run regularly, which will check for expiring certificates and renew them automatically.

We will run the certbot renewal command daily, using `cron`. To do so, we need to edit a file called a `crontab`.

```sh
$ sudo crontab -e
```

Paste the following line, then save and close the file.

```sh
15 3 * * * /usr/bin/certbot renew --quiet
```

Let's break that out and look at what the command is doing:

1. `15 3 * * *` - Run the following command at 3:15 am, every day. You may choose any time to run the command at.
2. `/usr/bin/certbot` - Calls the `certbot` utility.
3. `renew` - Tells Certbot to check all certificates installed on the system and update any that are set to expire in less than thirty days.
4. `--quiet` - Tells Certbot not to output information or wait for user input.

`cron` will now run this command daily. All installed cerificates will be automatically renewed and reloaded when there is thirty or less days before they expire.

##### Creating the ssl-defaults.conf snippet

In `/etc/nginx/snippets/`, create a file called `ssl-defaults.conf`, with the contents of:

```nginx
# These next two lines depend on where certbot placed your SSL certificates
# edit them to match the location of the files
# ssl_certificate /path/to/ssl/certificate
# ssl_certificate_key /path/to/ssl/certificate_key

ssl_dhparam /etc/ssl/certs/dhparam.pem;
ssl_prefer_server_ciphers on;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_stapling on;
ssl_stapling_verify on;
# The following setting can be dangerous, please research what it does before uncommenting.
# add_header Strict-Transport-Security max-age=15768000;
```

Then add `include snippets/ssl-defaults.conf` to your server block.

!! Note: These defaults may not fit your needs in every case. You may have to configure different settings in your server block.

! Also note that if you need to change these default settings, just remove `include snippets/<item>-defaults.conf` from the server block.

These default settings allow you to get an A+ from [SSLLabs](https://www.ssllabs.com/ssltest/index.html).

#### Default Server Block Template

Server blocks should be added to `/etc/nginx/sites-available`. The file should be named `<subdomain>.example.com.conf`.

```nginx
server {
    listen 443 ssl;
    server_name www.example.com;
    ssl on;
        # Remember to comment these out if you need to change their defaults
        include snippets/ssl-defaults.conf;
        include snippets/proxy-defaults.conf;

    location / {
        proxy_pass  http://<internal_ip>:<port>/;
        # Input any other settings you may need that are not already contained in the default snippets.
    }
}
```

After creating your server block, you need to link it to `/etc/nginx/sites-enabled/` using the following command.

```sh
$ sudo ln -s /etc/nginx/sites-available/www.example.com.conf /etc/nginx/sites-enabled/
```

Test the Nginx config using the following command:

```sh
$ nginx -t
```

Once you have configured all of these settings, you need to reload nginx, so that it has the new config.

```sh
$ systemctl reload nignx
```

You should now be able to reach your internal site from the internet.
