---
title: "GCP IPs All Appear in US, and It Has Been Marked as Intended Behavior"
date: 2020-03-22T23:12:59+08:00
draft: false
toc: false
images:
tags: 
  - gcp
---

As you may have already known, all the IPs of Google Cloud Platform share the same geolocation, which is the US. This buggy thing has been marked as "intended behavior", so no fix would be expected in the future. Even ironically, the guy who raised this issue was "kindly advised" to contact Google's legal department. Here is the [issue ticket](https://issuetracker.google.com/issues/112448138).

![You are kindly advised to contact Google's legal department](/img/you-are-kindly-advised-to-contact-googles-legal-department.png)

Dear Google, when people create a network in a chosen region, they expect the public IPs of the network to be in that region. Let's take the GDPR thing as an example. You set up a service in Europe region, but a geolocation lookup of the service IP shows it is in the US. Even though the service is indeed running in Europe region, you are gonna need some effort to convince your customers that there should be no trace that the data is going out of Europe.

I can tell another issue caused by this "feature". A few days ago, a friend of mine encountered a weird challenge. They have servers running in both Asia and North America, serving Asian and North American customers respectively. Recently they integrated a new third party service (let's call it TPS) to their backend services. This TPS is claimed to have replicas running in both Asia and North America. So requests originating from Asia should be hitting the TPS replica in Asia, and requests originating from North America should be hitting the TPS replica in North America. Out of expectation, all requests were going to the TPS replica in North America. This adds about 200ms extra latency to every customer request!

![Unexpected traffic routing](/img/unexpected-traffic-routing.png)

And there are other people facing the same issue.

[This Stackoverflow user](https://stackoverflow.com/questions/58108523/outgoing-http-request-location-on-google-app-engine) migrated his service from DigitalOcean to GCP and the service didn't work healthily, because "a third-party API only accepts requests coming from Brazil".

[This Stackoverflow user](https://stackoverflow.com/questions/51801691/created-gce-in-europe-region-but-ip-address-shows-its-in-us) gave up using GCP and turned to Azure, because "they have strict guideline not to transmit data out of Europe region and they couldn't prove to customers that their service is actually in Europe not in US".

Nowadays lots of existing infrastructure (firewalls, CDNs, etc) and services rely on IP geolocation. The "all IPs sharing same geolocation feature" for sure fools them all.

And I'm waiting for more victims.
