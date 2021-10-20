+++
title = "A server at home"
date = "2021-09-21T08:04:44-03:00"
draft = false
toc = true

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["tech", "en"]
+++

In the last few months I learned how to configure a Linux server from scratch, with no graphical interface, and I've been using applications hosted on it to make some things in my life easier. The reason for doing it this way - instead of paying for one of the many services that already exist - was purely out of a desire to learn, tinker, and make it happen.

This article is not intended to be a step-by-step tutorial on how to set up such a server, but you can use it as inspiration if you want to do it at your home. And of course, if you have any questions or want more details about something specific, be sure to contact me!

## Introduction

When I say I have a server, people who know what a server is instantly cringe. The servers used in businesses are really huge, heavy, and too noisy to have in a bedroom. Imagine sleeping with the sound of the cooler, the 24/7 heat and the flashing lights! But any computer can be a server for home purposes. The smaller the better! Here is my setup:

![Odroid HC1 next to a Pilot pen](/images/servidor-em-casa/img1.jpg)

I'm using an Odroid HC1, which has a VERY small board with only a few I/Os. It also has the space for a 2.5" HDD or SSD. I don't see many for sale here in Brazil, specially because it was [discontinued](https://www.hardkernel.com/shop/odroid-hc1-home-cloud-one/), but I was lucky to pay only R$220,00 (around 40 dollars) on MercadoLivre in mid June 2020.

The operating system runs from an SD card (which you need to flash in another computer beforehand), and when you power up the HC1 and plug in the network, it will receive an IP by DHCP. You then need to log into your router to get it's IP among the other devices and then connect via SSH.

Super easy until here when you take into account that it consumes less than 10 Watts in idle. There is also the concept of freedom, in the sense that my server is in my room and my services are 100% under my control. I plan to write an article about my relationship with digital privacy in the future, but just know that it is a big deal for me.

## Applications I'm using

From the beginning I already had an idea of which applications I would like to host on the server, and now, just over a year later, I have arrived at the following set of solutions, all of which are free (as in "free beer" and "freedom") and open source:

### [Kimai](https://www.kimai.org/)

This is a web application to register my working hours. I used to visit several customers throughout the day, so it was much better to control it here than in a Google Docs spreadsheet.

### [BookStack](https://www.bookstackapp.com/)

Another web app. It serves as a hierarchically organised wiki. Instead of thinking in pages with infinite subpages, it simplifies the organization with the following concept:

Shelves -> Books -> Chapters -> Pages.

I ended up using it only as a journal and to track some other personal stuff, since I didn't want to fill a personal application with work information. I didn't have the opportunity to implement it at the company because my colleagues use Microsoft solutions, but it's a good idea to document work environments. You can add multiple users in the system, control some permissions, comment on pages and see every page's revision history.

### [Firefly III](https://www.firefly-iii.org/)

This is a big darling of mine. I've been tracking my personal finances for four years now, starting with a Google Docs spreadsheet, which then moved to a paid system called Mobills. It was good for basic tracking, but I became averse to it when I realized that the data export tool did not export transfers between accounts (something I used to do a lot). Exporting only deposits and expenses would leave a big gap in my data in the long run. I ended up creating a JavaScript program to export my data (I plan to finish it and share it in the future) and migrated everything to Firefly.

I just had to learn a few new concepts - like the fact that credit cards are interpreted within the system as asset accounts that always go negative, and you need to make monthly transfers to represent the statements being paid. Personally I think it's very important to control this part of my life and I have nothing to complain about Firefly. It has good reporting features and, **mainly**, allows you to export all of your transactions, unlike Mobills.

### [Radicale](https://radicale.org/3.0.html)

This is a simpler application, which doesn't even have a web front-end. It's a CalDAV server to store contacts and calendars. Instead of handing over and relying on Google or any other service out there, I leave my contacts always under my control. Then I use an open source app like DAVx5 on my phone to sync with the server.

## How this works

All applications on this list work with a [LAMP stack](https://www.ibm.com/cloud/learn/lamp-stack-explained) (Linux, Apache, MySQL and PHP) except Radicale, which uses Python. The documentation for all services is very comprehensive and easy to follow. But this is where I need to add a *disclaimer*. With my recent interest in the DevOps culture, I wouldn't do this same setup if I was starting today. All four of these applications have Docker containers available, always well updated and documented. Spinning up a bunch of services entirely by hand like this has its dangers on an enterprise context - risks that are entirely avoidable if you know how to configure and follow good practices using containers.

However, it is clear that for a home server, it is not a problem to do things this way. In fact, it's even better to start it like this, because you really understand how a web application works - how to configure an Apache hosts file, how Apache will listen to ports, how to create and configure an SSL certificate, how to install PHP modules to fulfill the prerequisites, how a `.env` file works, how to create users in MySQL, how to backup everything etc.

## Moving forward

In the upcoming months I plan to study Docker and AWS. With that, it will be useful to spin up an EC2 instance (especially if it's elligible to the free tier) to learn the features offered by that platform. I'm thinking of dedicating an instance to replace my home server for a very simple reason: it is not possible to open the ports 80 and 443 in my ISP's router. With that, all my applications get a URL similar to the following: `https://domain.com:9011/kimai`. This is not an issue per se, except that it is not elegant **at all**. One way to solve this is using a [reverse proxy](/en/blog/reverse-proxy), but I don't use this solution myself because I am happy connecting to my network with a Wireguard VPN hosted on a router running [OpenWRT](https://openwrt.org/). Yep, I love this stuff. I plan to write an article about this, because I ran into problems configuring Wireguard and I think this will help some people.

I don't intend to retire the HC1 just yet - on the contrary: it's still extremely similar to the Raspberry Pi, which allows me to run several projects out there with good compatibility. Let's see where this will take me.