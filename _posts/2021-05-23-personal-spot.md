---
layout: default
title: Spot instances for personal servers
---

# {{ page.title }}

*{{ page.date | date_to_string }} on [Peter Vernigorov's blog](/)*

## 2023 Update

I have since moved from AWS to Hetzner as it is significantly cheaper than spot instances on AWS, at least for the workload that I have. 2 instances (`t3.nano` and `t3.medium`) with an EBS volumes were replaced by a single `CAX11` (2x arm64, 4GB RAM).

## Summary

Use spot instances for personal projects to save ~70% costs, and spot.io to ensure their stability.

## Why?

There was a recent post on lobste.rs asking ["What are you self hosting?"](https://lobste.rs/s/p4edt5/what_are_you_self_hosting_2021) and while some people prefer to host as little as possible relying on various services, many are running their own servers.

There are quite a few cheap/free options for compute instances. Some are listed [here](https://github.com/ripienaar/free-for-dev#major-cloud-providers). However, if one would like to stay on AWS, beyond 1 year of free 750 hours/month of micro instances, there isn't much that can be done to save costs.

In this post I describe my setup, that keeps the costs and maintenance as low as possible.

## Minimize complexity

While orchestration systems like Kubernetes can and do simplify the workflow, thereby improving delivery time and other KPIs, *they come at a cost*.

My employer has been an [early adopter of Kubernetes](https://github.com/zalando-incubator/kubernetes-on-aws), and now it is as easy to deploy a service as simply pushing to a branch. But it comes at non-zero maintenance costs. There are multiple teams responsible for maintaining and updating Kubernetes clusters, running CI/CD tools, documenting best practices, supporting teams in their work, etc. [See a good write up here](https://srcco.de/posts/how-zalando-manages-140-kubernetes-clusters.html).

For a personal project, unless the goal is to acquire experience with Kubernetes, it is usually a huge overkill both in terms of overhead and maintenance costs.

I run Amazon Linux 2 distributions, heavily utilizing systemd to run everything.

## Spot

Every personal server I have runs on a spot instance.

![Spot.io dashboard](/images/spot.png)

Spot instances are unstable by design, and can go down any time. However, in practice I have seen very few terminations. My longest uptime has been above *300 days* (in region eu-west-1), and the majority of interruptions was me restarting them for various reasons.

Running on spot trains one to expect terminations. After all, any instance, be it spot, on-demand, or even a server in your house, can be restarted or terminated at any time.

Additional benefit of being prepared for terminations is to be able to run instances only when needed. I have a beefy instance dedicated to hosting a Minecraft server, and it's only up when I need it. If I forget to turn it off, it automatically goes down at midnight.

This is where [spot.io](https://spot.io/) comes in. They help manage spot instances by foreseeing terminations based on spot market shortages, shutting the instance down safely, finding an instance type and an availability zone with better score, and starting your instance up again. They can manage 20 instances for free, which is enough for most people.

Each instance type in every availability zone gets a score. In case of a termination, spot.io will use a better scoring instance/AZ to boot the instance back up. I use `t(2,3,3a).(micro,small)` instances. In case of low scores across all instance types/AZs, there is an option to automatically go to on-demand until spot market improves.

![Spot Market Scoring](/images/spot-market.png)

In my experience, instance recycling takes about 5 minutes, which even with a daily termination would be sufficient for most personal project.

## Root Volume persistence

The killer feature for me is root volume persistence. My instances do not have any additional storage apart from the root EBS volume, which contains OS, all packages, and my applications. Usually, the root volume is deleted on termination, but spot.io creates an AMI from the root volume. So next time my instance boots up using that AMI, it will have all the same content.

## Public IP

There are two options for instance's public IP. For always on instances I use Elastic IP, which is free as long as the IP is attached to an running instance. And for instances that I only bring up when needed, I use an auto-assigned IP with a simple startup script that updates DNS. For Cloudflare I use the following:

`/home/ec2-user/update-dns.sh` (replace ZONE_ID, DNS_ID and API_TOKEN with correct values from the dashboard)

```bash
#!/bin/bash

IP=$(curl -s "http://169.254.169.254/latest/meta-data/public-ipv4/")

curl -X PATCH \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$DNS_ID" \
  -H "Authorization: Bearer $API_TOKEN" -H "Content-Type:application/json" \
  -d "$(cat <<EOF
{"content":"$IP"}
EOF
)"
```

`/usr/lib/systemd/system/update-dns.service`

```ini
[Unit]
After=network.service

[Service]
ExecStart=/home/ec2-user/update-dns.sh
User=ec2-user
Group=ec2-user

[Install]
WantedBy=default.target
```

## Non-24x7 instances

For managing instances that are up only when needed, I have 2 shell functions:

```bash
# Fish shell syntax, but can be easily changed to support bash/zsh

function spotstart
  curl -X PUT -s \
  -H "Authorization: Bearer $API_TOKEN" \
  "https://api.spotinst.io/aws/ec2/managedInstance/smi-$INSTANCE/resume?accountId=act-$ACCOUNT_ID"
end

function spotstop
  curl -X PUT -s \
  -H "Authorization: Bearer $API_TOKEN" \
  "https://api.spotinst.io/aws/ec2/managedInstance/smi-$INSTANCE/pause?accountId=act-$ACCOUNT_ID"
end
```

Alternatively, instance can be setup to be up only during specific days/hours:

![Run instance between 9-5 every weekday](/images/spot-cron.png)

## Fine print

I am not affiliated, associated, authorized, endorsed by, or in any way officially connected with spot.io, apart from using their services.

There are small additional costs associated with using spot.io:

- CloudWatch - spot.io collects data about the health of an instance using CloudWatch, and those API requests add up. With 2 instances, over a month I see about 6 thousand API requests (which are below the 1 million free requests) and 200 thousand metrics requested using GetMetricData API, which costs about $2/month.
- EBS snapshots - spot.io periodically creates a snapshot of EBS root volumes, which ends up costing additional ~50% of their normal cost. For 2 8GB root volumes this ends up being ~$0.87 in my region.

As can be seen, this is nowhere near close to savings from one instance alone, but worth being aware of.

***

Discussion: [lobste.rs](https://lobste.rs/s/ix7ozd)
