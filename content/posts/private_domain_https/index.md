---
title: "Private Intranet SSL certificate generation using DNS-01 challenges."
date: "2026-02-16"
tags: ['SSL', 'TLS', 'HTTPS', 'Self-Hosting', 'Intranet', 'CI/CD', 'CRON', 'DNS-01', 'Tutorial']
description: "Using a command-line based ACME client to automate the generation of SSL certificates for intranet servers."
author: ["Naveen Kannan"]
weight: 10
draft: true
ShowToc: true
TocOpen: true
cover:
  image: "https://letsencrypt.org/images/letsencrypt-logo-horizontal.svg"
  alt: "<alt text>"
  caption: "The Let's Encrypt logo."
  relative: false
  hidden: false
  hiddenInList: true
  hiddenInSingle: false
params:
  comments: true
  ShowCodeCopyButtons: true
  ShowReadingTime: true
---

# Introduction

When deploying internal services/tools on a private intranet, these services are usually hosted on servers that do not have a public IP address interface enabled, and only have a private IP address that only accepts connections through a firewall/IP whitelist. This approach ensures that these tools are only accessable internally, and are not public facing.

An approach like this, however, has a few caveats to be aware of. If you want to make requests to these services hosted on private servers, there are two options.

- Referring to the IP address directly
- Referring to a domain name. This requires that your DNS service is correctly configured to have the domain name resolve to the IP address associated with it.

Either way, without any further configuration, if you were to simply call these services either by the IP address or the domain name, you are greeted by warnings and disclaimers that state that your connection to this service is **insecure**. By default, these tools will not allow you to access these services without manually allowing the connection or by permitting the use of an insecure connection.

For example, if I was to host an API server on a VM with a private IP address of `10.0.0.1`, and accessed it as follows:

```bash
curl https://10.0.0.1/api
```
Or, if your DNS server resolves the domain name `dev.company.com` to the `10.0.0.1` IP address,

```bash
curl https://dev.company.com/api
```

This is the following output you can expect:

```bash
curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

These errors occur because these services expect a SSL certificate signed by a **trusted CA** by default. You will run into similar warnings when accessing front-end tools hosted on these servers as well.

There are a few options here.

- Use a self-signed SSL certificate.

This approach doesn't negate the warnings! A self-signed SSL certificate is still considered insecure by default.

- Spin up an internal CA

When you spin up an internal CA and generate certificates signed by this local CA, you will then have to **install this certificate** on the end user's devices, which might not be a feasible task. Certificates can expire, and you will have to automate the renewal, propagation and installation of the new certificates on the end-user devices on a regular basis.

The first approach doesn't fix the problem, and the second approach requires a lot of infrastructure and efffort to be built.

By far the easiest approach would be to have a **publicly signed SSL certificate**. This would remove the need to create your own internal CA service and the need to then install these local SSL certs on end-user devices.

However, this is where we run into the problem I'm hoping to address with this blog post. Globally trusted CAs usually **do not issue SSL certificates for private IP addresses**.

Even if you decided to issue an SSL certificate for your private domain name, you would need to create an external A record for it associated with your server's public IP address, which might be not be something you want to expose.

However, if your company already has a public domain name (which most companies do), with an associated DNS provider, it is entirely possible to have a publicly signed SSL certificate hosted on your private intranet server, without ever needing to expose your server's public IP address.

In this blog post, I will demonstrate how you can set this up and then automate the process of generating publicly signed SSL certificates for your private domain without the need for publicly exposing your server's IP address.

First, let's go over some basic fundamentals.

## Let's Encrypt and the ACME protocol

{{< figure
    src="https://letsencrypt.org/images/letsencrypt-logo-horizontal.svg"
    caption="The Let's Encrypt Logo."
    align=center
>}}

[Let's Encrypt](https://letsencrypt.org/) is a Certificate Authority that provides free TLS/SSL certiticates, and has been doing so for a full decade. It is a truly fundamental part of modern internet infrastructure, and is a project of the nonprofit Internet Security Research Group. Let's Encrypt automates the issual of certificates through an API that is based on the ACME protocol.

{{< figure
    src="https://upload.wikimedia.org/wikipedia/commons/thumb/1/16/ACME%E2%80%93protocol-icon.svg/500px-ACME%E2%80%93protocol-icon.svg.png"
    caption="The ACME protocol."
    align=center
>}}

The ACME (Automated Certificate Management Environment) protocol is a communication protocol that makes it possible to automatically obtain trusted certificates without human intervention, at very low cost. ACME is on version 2 of it's API, which was released on March 13, 2018, and this protocol was designed by the Internet Security Research Group for the Let's Encrypt Service.

Since Let's Encrypt requires the use of the ACME protocol, you will need an ACME client to interact with it programmatically.

## `dehydrated`, a bash based ACME client

{{< figure
    src="https://github.com/dehydrated-io/dehydrated/blob/master/docs/logo.png?raw=true"
    caption="The `dehydrated` Logo."
    align=center
>}}

[`dehydrated`](https://github.com/dehydrated-io/dehydrated) is a bash-based ACME Client that supports both the ACME v1 and ACME v2 protocols.

Lightweight (it's pretty much a bash script!) and simple with an MIT license, this is what we'll be using as our ACME client.

## ACME Challenges

When requesting a signed certificate from Let's Encrypt via an ACME client, Let's Encrypt vlidates that you control the domain name by issuing **challenges** defined by the ACME standard. There are three types of challenges, HTTP-01, DNS-01 and TLS-ALPN-01.

TLS-ALPN-01 is highly technical and uses a custom ALPN protocol, and isn't very relevant for this post, so we won't go over it this time.

### HTTP-01

This is the most common challenge type, where the client proves control over a domain name by proving that it can provision HTTP resouces on a server accessible under that domain name.

This challenge type requires the creation of A/AAAA records with your DNS provider. During the challenge process, Let's Encrypt servers will connect to at least one of the hosts found in the DNS A and AAAA records. For this challenge to successfully validate, the server must be reachable at the specified IP address.

This is why we won't be going with HTTP-01 challenges in this blog post, since that would defeat the purpose of having a private intranet to begin with.

>[!NOTE]
> DNS `A` records indicate the IP address of a given domain name, in the IPv4 format.
> DNS `AAAA` records indicate the IP address of a given domain name in the IPv6 format.

### DNS-01

DNS-01 challenges require the client to provision a `TXT` containing a designated value for a specific validation domain name.

After Let's Encrypt issues the ACME client a token as part of valdiation, the client will create a `TXT` record with your DNS procider derived from that token and account key. Let's Encrypt will then query the DNS system for that record, and will then issue a certificate if it finds a match.

This is the challenge type we will be using in this blog post, since this approach **does not** require us to expose the servers to the public internet. However, this approach requires that your DNS provider has API functionality to automate this process.

>[!NOTE]
> DNS `TXT` records store text information often used for verification.

# Prerequisites

With a brief explanation of the fundamentals completed, let's move forward with the prerequisites you will need before following along with this post.

You will need:

- A DNS provider with API functionality.

>[!TIP]
>While it is possible to manually update `TXT` records every 60-90 days, that approach is highly discouraged. ACME was built on the principle of automation,
> so if your DNS provider does not have API functionality, I highly recommend that you swap to a DNS provider that supports an API.
>
> The Let's Encrypt community has created a [resource to track DNS providers who provide API support](https://community.letsencrypt.org/t/dns-providers-who-easily-integrate-with-lets-encrypt-dns-validation/86438), which is a very useful resource.

- Your DNS server (different from your DNS provider) maps the IP address of your private server to the domain name you intend to use.

- cURL, sed, grep, awk, mktemp and `openssl` on your Unix OS.

## Installing the `dehydrated` client

The [`dehydrated`](https://github.com/dehydrated-io/dehydrated) ACME client can be installed as follows:

```bash
sudo su
# Downloading the Dehydrated client
cd /opt
git clone https://github.com/lukas2511/dehydrated.git

# Creating the necessary folders
cd dehydrated
mkdir certs accounts hooks
```

# Citations

- Bryan, B. (2016, September 12). **Intranet SSL Certificates Using Let’s Encrypt | DNS-01.** https://b3n.org/intranet-ssl-certificates-using-lets-encrypt-dns-01/

> [!NOTE]
> The inspiration and guide for this blog post is this [blog post by Benjamin Bryan](https://b3n.org/intranet-ssl-certificates-using-lets-encrypt-dns-01/) that was published on September 12, 2016.
> Ben's [blog is excellent](https://b3n.org/) and he writes on a lot of interesting topics.
>
> I highly recommend that you go check out his [original blog post,](https://b3n.org/intranet-ssl-certificates-using-lets-encrypt-dns-01/) which is what I followed while setting up the scripts and automations that I discussed in this blog.
>
>This post would have been impossible without the original subject matter as a guide!

# References

- The [Let's Encrypt Website.]()
- Let's Encrypt's [documentation on Challenge Types.](https://letsencrypt.org/docs/challenge-types/)
- [RFC8555 documentation on DNS-01 challenges.](https://datatracker.ietf.org/doc/html/rfc8555#section-8.4)
- [RFC855 documentation on HTTPS-01 challenges.](https://datatracker.ietf.org/doc/html/rfc8555#section-8.3)
- Let's Encrypt's [documentation on recommended ACME clients.](https://letsencrypt.org/docs/client-options/)
- The [ACME protocol on Wikipedia.](https://en.wikipedia.org/wiki/Automatic_Certificate_Management_Environment)
- The [dehydrated ACME client.](https://github.com/dehydrated-io/dehydrated)
