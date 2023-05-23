---
layout: post
title: Home Intrusion Detection and Monitoring with ARP Poisoning
description: A walkthrough on a home IDS solution done through ARP poisoning.
summary: A walkthrough on a home IDS solution done through ARP poisoning.
tags: projects writeups ids ips
minute: 10
---

# Overview
I had planned out to create a home IDS/IPS solution on my home network with a PFSense firewall. However, because my internet provider gave me the ARRIS BGW320-500 model for fiber connectivity, that would not be possible.

I did do some research and found an alternative that did not require me to purchase additional hardware or opt into cloud networking: ARP cache poisoning.

It makes perfect sense when I found out about it. If we can intercept all the network traffic meant for the gateway, we would be able to sniff the interface without using a network TAP. One great tool for this is [bettercap](https://github.com/bettercap/bettercap.git).

So for now, my setup includes:
- A cheap managed switch
- RITA/Zeek/MongoDB
- bettercap
- ELK Stack
- an old server

I purchased a [cheap managed switch](https://www.amazon.com/dp/B07BNVTZ3S?psc=1&ref=ppx_yo2ov_dt_b_product_details) for around $20. The way my cabling to my gateway is setup, a switch is absolutely necessary for me if I need a hard-wired connection.

# The Design and How It Works
![](/assets/projects/homelab/diagram.jpg)

I also have an old but solid server that has an Intel i5 chip and installed Ubuntu 20.04 (That was the only ISO I had).

## Installing and Setting Up ELK Stack
![](/assets/projects/homelab/kibana.png)
>server.host and network.host is set to 0.0.0.0 so I can access it from another machine. Localhost is limited to the host.

![](/assets/projects/homelab/elastic.png)
Once those are configured, we'll start all three services (logstash, elasticsearch, and kibana)
![](/assets/projects/homelab/dashboard-elastic.jpg)

## RITA
RITA is short for Real Intelligence Threat Analytics. https://github.com/activecm/rita has a install script to download the necessary dependencies for the binary, including Go. It also includes Zeek and MongoDB.

I installed from the sources (for some odd reason), but it was a process so I won't cover the installation process.

## bettercap
To install BetterCap, you need the 'make' package and Go. Once those two are present, clone the repository and install using:
{% highlight shell %}
git clone https://github.com/bettercap/bettercap.git
make install
{% endhighlight %}

To get started using BetterCap's web UI, you will need to run : `sudo bettercap -eval "caplets.update; ui.update; q"` for the first and last time.

Because I am wanting to access the service from another workstation, I will run bettercap using it's https-ui module as it is more secure. But first I need to modify the default credentials in `/usr/local/share/bettercap/caplets/https-ui.cap`
![](/assets/projects/homelab/bettercap.png)
>Very important to change the default credentials.

Once that's changed, I can run it using `sudo bettercap -caplet https-ui`.
![](/assets/projects/homelab/bettercap2.png)

If you get a TLS certificate error when connecting, you will need to add an exception for the right port. Bettercap runs off of http.server and api.rest and your browser will only catch http.server.

I solved this issue by going to the Certificate Manager in Firefox and adding an exception to the rest.api port 8083.
![](/assets/projects/homelab/cert.jpg)

And now I should be able to log in:
![](/assets/projects/homelab/bettercap3.jpg)

# Monitoring and Logging the Network
Now that we've gotten everything installed and running, let's configure our tools so that we can capture the network traffic and ingest it into our SIEM solution.

## Configuring Zeek
We want to run Zeek as a service so that all the logs can be processed into our ELK instance. First we need to change the log format to .json instead of TSV in the `/opt/zeek/share/zeek/site/local`.zeek file.
![](/assets/projects/homelab/zeek.png)

We'll also identify what interface we're sniffing traffic from in the `/opt/zeek/etc/node.cfg` file. In my case, it's eno1.
![](/assets/projects/homelab/zeek2.png)

And to ensure that Zeek can capture packets, we'll run these two commands:
{% highlight shell %}
sudo setcap cap_net_raw=eip /opt/zeek/bin/zeek
sudo setcap cap_net_raw=eip /opt/zeek/bin/capstats
{% endhighlight %}

And finally we add Zeek to our $PATH:
{% highlight shell %}
export PATH=$PATH:/opt/zeek/bin
{% endhighlight %}

And we can run `zeekctl start` to see if it will run:
![](/assets/projects/homelab/zeek3.png)
>It runs!

## Configuring Filebeat
We will be using Filebeat's Zeek module to ingest our Zeek logs into Elastic. It doesn't come included with the ELK stack, so we'll install it on the host.

After installing it, we need to enable the Zeek module with `sudo filebeat modules enable zeek`.

A new zeek.yml file should be created in `/etc/filebeat/modules.d/`. The .yml file contains different types of Zeek logs and gives us options to enable/disable certain logs from being ingested into our SIEM. We're essentially monitoring a directory to pull logs from.

We need to edit those directories into the .yml file. Here's what mine looks like:
![](/assets/projects/homelab/filebeat.png)

You simply add another field under each log type called `var.paths` and supply the path to the log. If there are log files we do not want, we can simply switch them to false and leave it alone.

Once we are content with the configuration, we can run it with:
{% highlight shell%}
sudo filebeat setup
sudo service filebeat start
{% endhighlight %}

# Visualizing Everything
Now that we've got everything configured, we can start capturing traffic and process them with our tools.

## RITA
Whenever we want to capture any network traffic in our local network, we'll want to run the arp.spoof module in bettercap.

Note that in order for the logs to be effective, change `arp.spoof.fullduplex` to `true`. This value tends to default to false.
![](/assets/projects/homelab/bettercap4.jpg)
>Click arp.spoof run to begin the module

I generated a PCAP file from tcpdump (Tshark works fine as well) and processed it with Zeek.
![](/assets/projects/homelab/rita.png)
![](/assets/projects/homelab/rita2.png)

Then to process all the Zeek logs into RITA, we need to use:
{% highlight shell %}
rita import path/to/your/zeek_logs dataset_name
{% endhighlight %}

If we plan on using the same set of logs and want to update the same dataset, we would use the `--rolling` switch.

If there's an error saying that the database is unreachable, check to see if the mongod service is active.

We can create an HTML report using `rita html-report` and it will place the files in a directory with the same name as your dataset.
![](/assets/projects/homelab/rita3.png)

## Elastic
Since we configured Filebeats with the Zeek module, any logs that exist in the `/opt/zeek/logs/current/` directory should be ingested into Elastic.

So all that means is all we need to do is enable ARP spoofing on bettercap and start zeekctl.

After we confirm those are running, if we look into the Discover tab on Kibana, we can see an index called `filebeat-*`.
![](/assets/projects/homelab/discover.jpg)

In the Dashboards tab, if we search for `Zeek`, we can find a dashboard called `[Filebeat Zeek] Overview`.

Click on it, and we are met with a beautiful dashboard that visualizes our network traffic.
![](/assets/projects/homelab/dashboard.jpg)
![](/assets/projects/homelab/dashboard2.jpg)

# Conclusion
And that sums up my network intrusion detection and monitoring lab in my home network.  I am planning on adding Suricata in the near future. I've been playing with Suricata in a virtual environment and found that we can integrate Wazuh into Elastic and ingest alerts that way. Here are some ideas that I plan on implementing:
- DNS Pi Hole
- Suricata rules
- Maybe SecurityOnion..?
- and more!

This is still a work in progress and I hope to finish it soon so that I can fully walk through what I envisioned it to become.