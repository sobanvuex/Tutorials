How To Use Nginx as Global Traffic Director on Debian/Ubuntu
------------------------------------------------------------

`Nginx`, `DNS`

> Global Traffic Directing is a way to redirect users to the closest geographically located server.
This way users from Asia would be served by servers in Asia and users in Europe by servers in Europe.

> Most if not all major DNS providers support this feature, but it sometimes is too expensive.
This tutorial will explain how you can setup "poor man's GTD" using Nginx, GeoIP and Subdomains.


### Introduction

So often you have a website which is hosted in one location, as your customer base grows so does the distance between your server(s) and your customers. We all know that if your server load increases - you scale. But what to do when the distance is the problem. Well the solution is simple - install server(s) in geographical locations closer to your customer base and direct them based on their location. But how to do this easy and cheap? Let me show you.

In this guide we're going to configure Nginx to detect and redirect customers to a subdomain pointing to a geographically more suitable server.

### Prerequisites

To complete this guide you will need a user with `sudo` privileges. One or more servers, where the additional ones are in different regions.

### Assumptions

In this tutorial we are going to have a few assumptions:

- Your domain is `www.example.com`
- Your primary server is in US
- You want to install a GTD for Europe and Asia
- Your server IPs are as follows:
  - US: `1.1.1.1`
  - EU: `1.1.1.2`
  - AS: `1.1.1.3`

### Step 1 - Subdomains and DNS configuration

Choosing subdomains is all up to you. For this tutorial lets use `na.example.com` for US, `eu.example.com` for Europe and `as.example.com` for Asia.

For each of those subdomains add a `A record` in your DNS configuration with the IP of the server for that region:
- `na.example.com` - `1.1.1.1`
- `eu.example.com` - `1.1.1.2`
- `as.example.com` - `1.1.1.3`

It should look something like this:

![dns](https://cloud.githubusercontent.com/assets/711758/2891198/6098696c-d530-11e3-9ef3-fee7a1ac37d5.jpg)

### Step 2 - Install Nginx and GeoIP

To have Nginx with GeoIP module you have two options - use a precompiled package (only -full and -extra have GeoIP module) or compile your nginx with the `--with-http_geoip_module` configuration parameter (In this case you also need geoip-dev libraries).

Lets use the already available repository packages

```
sudo apt-get update
sudo apt-get install nginx-full geoip-database
```

Great now both Nginx and GeoIPs binaries are available. But there is one thing more. GeoIP's city database, which includes region information. You need to download and install it manually

```
wget -N http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz
gunzip GeoLiteCity.dat.gz
mv GeoLiteCity.dat /usr/share/GeoIP/
```

### Step 3 - Configure Nginx and your virtual host

Here we are going to tell Nginx where the GeoIP database files are. And in our virtual host we are going to configure how Nginx to respond to request based on their geoip information.

Open your `nginx.conf` (default `/etc/nginx/nginx.conf`) with your preferred editor. Add the line `geoip_city /usr/share/GeoIP/GeoLiteCity.dat;`. Your `nginx.conf` would look like this:

```
http {

  geoip_city /usr/share/GeoIP/GeoLiteCity.dat;

  ...

}
```

Save it.

Now lets edit your virtual host (default `/etc/nginx/sites-available/default`). Inside it we need to create a `map` and add the subdomains to the `server_name` directive.

The `map` in Nginx allows us to set a variable `$closest_server` based on the value of `$geoip_city_continent_code`. You can read more about the [Map module](http://nginx.org/en/docs/http/ngx_http_map_module.html) on Nginx's documentation.

```
map $geoip_city_continent_code $closest_server {

  default www.example.com;

  NA      na.example.com;
  EU      eu.example.com;
  AS      as.example.com;

}
```

Next we add the location based subdomains to the `$server_name` directive:

```
server {

  server_name example.com
              www.example.com
              na.example.com
              eu.example.com
              as.example.com;

  ...

}
```

The last part of the process is to make a condition in your virtual host to redirect visitors to the closest server. Add the following condition to your configuration

```
server {

  ...

  if ($closest_server != $host) {
    rewrite ^ $scheme://$closest_server$request_uri break;
  }

  ...

}
```

After you're done with all the changes your virtual host file would look like this:

```
map $geoip_city_continent_code $closest_server {

  default www.example.com;

  NA      na.example.com;
  EU      eu.example.com;
  AS      as.example.com;

}

server {

  server_name example.com
              www.example.com
              na.example.com
              eu.example.com
              as.example.com;

  if ($closest_server != $host) {
    rewrite ^ $scheme://$closest_server$request_uri break;
  }

  ...

}
```

** Repeat this step for each server you want to configure. That way any server will will act as a traffic director. **

### Step 5 - Run a few tests

After completing all the step the final one is to test what you have done. When working with Nginx always test new configuration before applying. Nginx provides an option to test its configuration files, without affecting currently running Nginx. You can do that by running one of these commands (they do the same thing):

`nginx -t`, `service nginx configtest` or `/etc/init.d/nginx configtest`

If everything is good - Reload your Nginx configuration:

`nginx -s reload`, `service nginx reload` or `/etc/init.d/nginx reload`

To see your traffic director in action. Open a browser and visit `www.example.com`:

![www](https://cloud.githubusercontent.com/assets/711758/2891968/f44a1456-d537-11e3-921b-8cb1afb34229.jpg)

In my case (Canada) I am immediately redirected to `na.example.com`:

![na](https://cloud.githubusercontent.com/assets/711758/2891967/f4435486-d537-11e3-80af-e35986515074.jpg)

Using a VPN/Proxy of choice I advise you to test your other regions as well.

And from now on your global visitors will be immediately redirected to a server close to them, improving their experience on your website.
