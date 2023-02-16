+++
title = "Using NGINX as a reverse proxy"
date = "2021-10-27T10:00:00-03:00"
draft = false
toc = true
# comments = true
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["tech", "aws", "linux", "en"]
+++

This is a guide on how to use NGINX as a reverse proxy. This is going to be a bit lengthy, so I recommend using the table of contents above if you want to skip to a specific step.

## Requirements

**Minimum**  
- A server running Linux

**Additional**  
- An AWS account
- A domain
- A [FreeDNS](https://freedns.afraid.org/) account

## What does a reverse proxy do?

![Diagram of a reverse proxy](/images/proxy-reverso/img22en.png)

A reverse proxy server is responsible for receiving HTTP requests (like `app2.domain.com` in the diagram above). When this request is in the list of servers that the reverse proxy recognizes, it redirects the traffic somewhere else. In this example, the `app2.domain.com` request would be redirected to port `10012` in the `srv.domain.com` server.

A practical use for this is when you have a server running several Docker containers. On another server, you can have a reverse proxy that will accept requests on different subdomains, redirecting each one of them to the corresponding containers on the first server.

It is also possible that the reverse proxy is on the **same** server as the application. In this case, the ports used by the applications are restricted for internal access only, and the reverse proxy takes care of staying at the edge waiting for connections and redirecting accordingly.

In this article, I'll give examples of the two setups: using one or two servers.

## More real world examples before we begin

You may want to use a reverse proxy when setting up a [home server](/en/blog/home-server/). Most ISPs prevent the opening of certain ports on the router provided for residential customers. Even if you can get into the setup page and create the virtual hosts, the ports remain closed. This happened to me when I was dealing with Vivo here in Brazil and it caused a lot of frustration. The ports I tested were 80, 443 and 8080, the first two being the most important when you are trying to host a web service. In this case, you can open non-standard higher numbered ports (such as ports above 1024), serve the pages over those ports and use a virtual machine on [AWS' Free Tier](https://aws.amazon.com/free/) as a reverse proxy. What the reverse proxy will do is intercept external traffic and redirect it to the server at home.

---

Another situation I faced was in a corporate environment. I worked for a company that provided solutions hosted on their own infrastructure to customers. These solutions were hosted on web servers, which customers accessed through URLs similar to the ones below:

```
https://portal.exemple.com:8080
https://cloud.exemple.com:9009
https://demo.exemple.com:8443
```

All of these being CNAMEs pointing to the same host. This leads to two issues: the presence of the port number in the URL looks very sketchy - personally it bothers me a lot to visit an URL like this; and it also just adds up to general mess and confusion the more the services grow.
Being a corporate customer, the company had total freedom to use ports 80 and 443, and let a reverse proxy server handle these redirections internally.

## Setup using just one server

### Local services

For demonstration purposes, I will host two Python-based applications. One of them will be a simple hello world in [Flask](https://flask.palletsprojects.com/en/2.0.x/quickstart/) and the other will be a comment system called [Isso](https://posativ.org/isso/) (the same one used on this website). The ports used will be `5000` and `8011` respectively. Here's what happens when I access the applications directly using IP and port:

![Browser displaying web pages on the local server](/images/proxy-reverso/img23.png)

Everything works just fine, but we want to avoid having to type the server IP and port to access the pages. Your particular situation will be different, but I suppose it's the same scenario: applications being served on several different ports. Let's get to the reverse proxy.

### Installing NGINX and configuring the reverse proxy

I am using CentOS 7. Before installing NGINX, I need to install the `epel-release` package because the default sources do not include the NGINX package. You may research for the specific details if you use another distribution. Then install the NGINX package:
```
$ sudo yum install nginx
```

Next up, we will enable the NGINX service so that it starts on system boot.
```
$ sudo systemctl enable nginx
Created symlink from /etc/systemd/system/multi-user.target.wants/nginx.service to /usr/lib/systemd/system/nginx.service.

$ sudo systemctl start nginx

$ sudo systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2021-10-07 15:07:34 UTC; 14s ago
  Process: 9257 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 9255 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 9254 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 9259 (nginx)
   CGroup: /system.slice/nginx.service
           ├─9259 nginx: master process /usr/sbin/nginx
           └─9261 nginx: worker process
```

---

According to the default NGINX configuration file (located in `/etc/nginx/nginx.conf`), the service looks for additional configuration files with the `.conf` extension in the `/etc/nginx/conf.d/` directory. That's where we'll create a file for each site.

We'll create `/etc/nginx/conf.d/flask.conf` and `/etc/nginx/conf.d/isso.conf`. The contents of these files will be, respectively:
```
server {
    listen 80;
    server_name flask.pckcml.test;
    location / {
        proxy_pass http://localhost:5000;
    }
}
```
```
server {
    listen 80;
    server_name isso.pckcml.test;
    location / {
        proxy_pass http://localhost:8011;
    }
}
```

Note that the `server_name` parameter in both files end in `pckcml.test`. I have a router that allows me to create DNS records, so I pointed them both to the IP of that server. Like this:

![Local DNS records](/images/proxy-reverso/img24.png)

In your case, you may have an external domain. Use the DNS panel of that domain to create A hosts or CNAMEs accordingly.

Every time you change the NGINX files, run the command below to test the settings and restart the service to apply what was changed:
```
$ sudo nginx -t && sudo systemctl restart nginx
```

### Testing the access to the services using the new address

If everything went right, I'll be able to access the URLs `flask.pckcml.test` and `isso.pckcml.test`, like this:

![Browser displaying web pages on local server](/images/proxy-reverso/img25.png)

This is the simplest NGINX setup, using a single server for both application and reverse proxy. The setup with two servers is slightly different.

## Setup using two servers

For this section, I will host some web pages on my home server. After that, I'll use a [virtual machine on AWS](/en/blog/aws-virtual-machine) to host the reverse proxy. I'll also use my domain's DNS.

One note before we go any further: Keep in mind that when you are using a VM on AWS as a reverse proxy, all your traffic pass *through* it. This isn't a problem when you host simple web applications, but if you start generating a lot of traffic, you can get billed quite a bit. This makes it too expensive, for example, if you are setting up a file server at home, as it tends to generate a much higher volume of data than the free tier on AWS allows. Be careful not to generate unexpected costs!

### Local services

For demonstration purposes, I will host two applications: [Kanboard](https://kanboard.org/) on port `10011` and a simple page I made with HTML on port `10012`. In this application server I am going to use Apache, for no particular reason other than diversity. The Apache `.conf` files for these applications look like this:

```
<VirtualHost *:10012>
        DocumentRoot /var/www/t1_colors
</VirtualHost>
```

So when I access `http://192.168.25.150:10012` I see the following page:

![Browser displaying web pages on the local server](/images/proxy-reverso/img2.png)

### Configuration for external access

After the application is working locally, it is necessary to open the ports on the ISP's router (in my case, Vivo) and test for the external access. My table of virtual servers on the Vivo router looks like this:

![Table of virtual servers on the Vivo router](/images/proxy-reverso/img3.png)

I am also using [FreeDNS](https://freedns.afraid.org/) so that I don't need to reach my server by IP. I created a FreeDNS entry with the address `patrickcamillo-rp-home.my.to` pointing to my external IP. Then I accessed the URL `http://patrickcamillo-rp-home.my.to:10011` through 4G on my phone, to make sure it's working externally. It looks like this:

![Browser displaying webpage on mobile](/images/proxy-reverso/img4.jpg)

### NGINX setup on the AWS virtual machine

The NGINX installation in the VM follows the same method indicated [earlier in this article](#installing-nginx-and-configuring-the-reverse-proxy). But the `.conf` files of the sites must point to my home server, not to `localhost`. The concept is the same:

```
server {
    listen 80;
    server_name rp-kanboard.patrickcamillo.com;
    location / {
        proxy_pass http://patrickcamillo-rp-home.my.to:10011;
    }
}
```
```
server {
    listen 80;
    server_name rp-colors.patrickcamillo.com;
    location / {
        proxy_pass http://patrickcamillo-rp-home.my.to:10012;
    }
}
```

Note that there are two `server_name` entries: `rp-kanboard` and `rp-colors`. I created both as A hosts in my DNS, like this:

![Route53, DNS entries created](/images/proxy-reverso/img15.png)

Before finishind and actually accessing the pages, I faced a small problem. Let's analyse and solve it.

### Possible issue with NGINX

If you look again at the diagram shown at the beginning of the article, you'll notice that the proxy server (the one with NGINX) needs to forward data to an external server (the one with the applications themselves). However you may come across this error screen when trying to access the proxy server:

![Error "502 Bad Gateway" in NGINX](/images/proxy-reverso/img16.png) 

At least that's what happened to me! In case it happened to you too, see the investigation I did below. The solution is at the end of this section.

---

I logged in as `root` to make it easier to read the NGINX logs, located in `/var/log/nginx/`. The first file I investigated was `access.log`. I ran the command `tail -f -n 0 /var/log/nginx/access.log`, and after that I refreshed the web page. I saw the following result in the terminal:
```
<MY_IP> - - [07/Oct/2021:20:42:02 +0000] "GET / HTTP/1.1" 502 157 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:93.0) Gecko/20100101 Firefox/93.0" "-"
```
Everything's fine here. The server knows that I'm requesting the page.

On the other hand, I saw the following message in the `error.log` file:
```
2021/10/07 20:43:12 [crit] 9685#9685: *77 connect() to <MY_SERVER_IP>:10012 failed (13: Permission denied) while connecting to upstream, client: <MY_IP>, server: rp-colors.patrickcamillo.com, request: "GET / HTTP/1.1", upstream: "http://<MY_SERVER_IP>:10012/", host: "rp-colors.patrickcamillo.com"
```

We have something! Seeing an error message is always better than seeing nothing. I did some research and found [this post on the NGINX blog](https://www.nginx.com/blog/using-nginx-plus-with-selinux/). It tells us that the problem involves **SELinux**. I'm not ashamed to admit that at the time of writing this article, I'm not very knowledgeable about how SELinux works. But the post already gives us a good introduction:

>SELinux is enabled by default on modern RHEL and CentOS servers. Each operating system object (process, file descriptor, file, etc.) is labeled with an SELinux context that defines the permissions and operations the object can perform. In RHEL 6.6/CentOS 6.6 and later, NGINX is labeled with the `httpd_t` context.

Ok, cool. With this, we've learned briefly how SELinux works and how exactly it is acting on NGINX. The article goes on to explain that SELinux grants various permissions for NGINX to work, but "proxying to upstream locations or communicating with other processes through sockets" is not allowed (in the sense that NGINX cannot, as a client, communicate with an external server).

The article shows how it is possible to disable the SELinux restrictions on NGINX temporarily or permanently. But this is not good, because it can open unnecessary and dangerous security holes. In no context is it interesting to disable security completely. We can investigate precisely which SELinux parameter is preventing the connection and change *only* this parameter.

I ran `tail -f -n 0 /var/log/audit/audit.log` and refreshed the page again. One of the messages that appear in the terminal is the following:
```
type=AVC msg=audit(1633647336.240:1355): avc:  denied  { name_connect } for  pid=9685 comm="nginx" dest=10012 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

We can search for the explanation of this message on the system itself, using the `audit2why` package. I ran this command: `grep 1633647336.240:1355 /var/log/audit/audit.log | audit2why`. Its output is as follows:
```
Was caused by:
One of the following booleans was set incorrectly.
Description:
Allow httpd to can network connect

Allow access by executing:
# setsebool -P httpd_can_network_connect 1
Description:
Allow nis to enabled

Allow access by executing:
# setsebool -P nis_enabled 1
```

The first command suggested by the tool is meant to enable a variable called `httpd_can_network_connect`. It "allows HTTP scripts and modules to initiate a connection to a network or remote port", definition that I adapted from [this documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-managing_confined_services-the_apache_http_server-booleans).

So all we need to do is run the command `setsebool -P httpd_can_network_connect 1` and then we can move on to the next step.

### Testing the access to the services using the new address

Finally, I could access the `http://rp-colors.patrickcamillo.com/` page and see the following result:

![Final result: Reverse proxy working for the page "colours"](/images/proxy-reverso/img17.png)

Same for page `http://rp-kanboard.patrickcamillo.com/`:

![End result: Reverse proxy working for page "kanboard"](/images/proxy-reverso/img18.png)

## Conclusion

This article taught how a reverse proxy works and how to set one up using NGINX. We also learned together the basic operation of SELinux. Both tools have several guidelines and configurations that are well worth researching and understanding how they work.

If you already have some knowledge of web servers, you noted that I did not use HTTPS, even in a service exposed externally. I'm saving that for an article in which I'll delve deeper into some NGINX configurations, as well as the part about generating a free SSL/TLS certificate with Let's Encrypt. Stay tuned for more!

In any case, thank you very much for following along so far. As always, if you have any questions, be sure to use the comments below. Thanks a lot!

---

## References

[Getting started with self hosting - episode 4](https://mickael.kerjean.me/2018/03/14/getting-started-with-selfhosting-episode-4/)  
[NGINX Reverse Proxy Documentation](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)  
[Using NGINX and NGINX Plus with SELinux](https://www.nginx.com/blog/using-nginx-plus-with-selinux/)  
[SELinux User's and Administrator's Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-managing_confined_services-the_apache_http_server-booleans)