---
layout: post
title: 'Ubiquiti EdgeRouter Lite Setup Part 5: OpenVPN Setup'
author: Seth Forshee
tags: 
---

In previous posts we've covered everything required to set up a network with
multiple VLANs and IPv6 (see [part 1]({% post_url 2016-03-01-ubiquiti-edgerouter-lite-setup-part-1-the-basics %}) for a list of all posts in this
series). Today we're going to talk about setting up an OpenVPN server on the ERL.

If you aren't familiar with VPNs, think of them as encrypted tunnels used to
connect computers over the internet. Common uses for VPNs include:

- Providing privacy while using public networks, especially open public wifi
  hotspots.
- Enabling secure, remote access to LANs, connecting remote networks.
  Corporations often use this to allow employees to access the corporate
  network remotely.
- Connecting networks in multiple, fixed locations. For example a business with
  multiple offices might use this to securely connect together the various
  office networks via the internet.

The VPN setup described here can be used for the first two use cases above but
not for the third.

There are several protocols which can be used to set up a VPN, including
[PPTP](https://en.wikipedia.org/wiki/Point-to-Point_Tunneling_Protocol),
[L2TP](https://en.wikipedia.org/wiki/Layer_2_Tunneling_Protocol),
[SSTP](https://en.wikipedia.org/wiki/Secure_Socket_Tunneling_Protocol), and
[OpenVPN](https://en.wikipedia.org/wiki/OpenVPN). I've chosen OpenVPN here
because it's secure, flexible, and open source. The paticular configuration is
very specific to my needs and level of paranioa. Consult the
[OpenVPN documentation](https://openvpn.net/index.php/open-source/documentation.html)
to help you modify this setup for your needs.

# Generating Certificates

OpenVPN uses public key cryptography in essentially the same way it's used to
make secure connections to websites. This means we need a public key
infrastructure capable of generating signed public/private key pairs, which in
turn means we need to create our own certificate authority (CA). There's a
script on the ERL to help us do that.

{% highlight console %}
$ sudo -s
# cd /usr/lib/ssl/misc
# ./CA.sh -newca
{% endhighlight %}

You'll be asked a number of questions during the process, and skipping them may
cause it to fail. Just answer the best you can. This is going to happen several
times while setting up the certificates.

If you supply a password here, make sure you remember it. You're going to need
it! Of course if you forget it you can always start the process over again at
the beginning.

Once this completes you'll have a `demoCA` directory containing a number of
files. The important ones are `cakey.pem`, which is the private key for your
CA, and `cacert.pem`, which is the public key.

Next we'll generate a public/private keypair for the OpenVPN server.

{% highlight console %}
# ./CA.sh -newreq
# ./CA.sh -sign
{% endhighlight %}

This will generate two files, `newkey.pem` and `newcert.pem`, which are
respectively the private and public halves of your server certificate.

Now we're going to move all of these key files to `/config/auth/`. This will
ensure that they're preserved across firmware upgrades and include in your
configuration backups.

{% highlight console %}
# cp demoCA/cacert.pem demoCA/private/cakey.pem /config/auth
# mv newcert.pem /config/auth/host.pem
# mv newkey.pem /config/auth/host.key
{% endhighlight %}

Next we'll generate a Diffie-Hellman parameter file. This will allow clients
and the server to generate shared session keys without ever having to transmit
that key over the internet, so even if someone compromised the server
certificate they would be unable to decrypt session traffic. I use a key size
of 2048, though 1024 is more common and probably safe (though I don't claim to
be a crypto expert).

{% highlight console %}
# openssl dhparam -out /config/auth/dhp.pem -2 2048
{% endhighlight %}

This is going to take a while, so go get a cup of your favorite beverage. No
need to hurry.

Once that's completed you'll need keys for clients, preferably one set per
client.

{% highlight console %}
# ./CA.sh -newreq
# ./CA.sh -sign
# mv newcert.pem /config/auth/client1.pem
# mv newkey.pem /config/auth/client1.key
{% endhighlight %}

At this point I note that some other guides state that you should remove the
password from your client key files. I honestly can't remember whether or not I
did this, or maybe I just didn't supply a password for the client certificates.
Anyway, here's the command to do this if you're having issues because the
client keys have a password, or if you're just getting annoyed at entering the
password each time you connect:

{% highlight console %}
# openssl rsa -in client1.key -out client1_nopass.key
{% endhighlight %}

You'll need to copy the `.pem` files to the respective clients to allow them to
make connections.

# OpenVPN Server Setup

Now it's time to set up the OpenVPN server on the ERL. This is done by creating
a new interface. You'll also need a new IPv4 subnet for the VPN; I use
192.168.200.0/24 here.

You'll also need to make decisions about which port to use, whether to use tcp
or udp, which routes to push, etc. For this example I'll use tcp on port 443,
which sort of disguises the traffic as normal SSL (useful because SSL
connections are never blocked by network admins). I'll push a route to allow
communication with clients in the office network.

_Update_: As pointed out in the comments port 443 conflicts with using SSL for 
the web gui. Your options to solve this are to either use a different port for 
the web interface (the `serivce gui https-port` setting controls this) or a 
different port for the VPN.

{% highlight console %}
$ configure
# edit interfaces openvpn vtun0
# set description OpenVPN
# set mode server
# set local-port 443
# set protocol tcp-passive
# set server subnet 192.168.200.0/24
# set server topology subnet
# set server push-route 192.168.103.0/24
# set tls ca-cert-file /config/auth/cacert.pem
# set tls cert-file /config/auth/host.pem
# set tls dh-file /config/auth/dhp.pem
# set tls key-file /config/auth/host.key
{% endhighlight %}

# Firewall Setup

Now we need to set up a firewall zone for the VPN and write rules for this
zone. In this setup the VPN is really just an extension of the office LAN, so
for the most part we can just reuse the same rules used for the office LAN
zone. There are a handful of cases where this isn't possible though:

- WAN to local: A rule is needed here to allow incoming tcp connections on port 443.
- VPN to office LAN: All traffic is allowed.
- Office LAN to VPN: All traffic is allowed.

Refer back to
[part 2]({% post_url 2016-03-02-ubiquiti-edgerouter-lite-setup-part-2-firewall-setup %})
for help setting up the firewall.

# Final Steps and Testing

The OpenVPN server hands out IP addresses to clients, so there's no need to set
up DHCP for the VPN subnet. You may or may not want to set up DNS to listen on
the vtun0 interface, depending on your needs.

Now it's time to save everything and try it out.

{% highlight console %}
# commit
# save
{% endhighlight %}

You'll need to configure your client machine(s) to connect using the client
certificate(s) we generated earlier. The steps for doing this vary depending on
your OS; please consult the
[OpenVPN documentation](https://openvpn.net/index.php/open-source/documentation.html)
for help.

For testing it's ideal to try and connect from outside your network, e.g. by
tethering to a phone. But for this setup it's also possible to test by trying
to use the VPN to access the office LAN from the home LAN. Just be sure you've
set up firewall rules to allow clients on the home LAN to connect to the
OpenVPN server on the router.

# Hardening

This section isn't essential, but I do recommend it.

The
[OpenVPN hardening page](https://community.openvpn.net/openvpn/wiki/Hardening)
covers various ways to improve the security of OpenVPN. It's useful to read
through these. The only one that I'm going to cover here is TLS auth.

The TLS auth option is pretty cool. It makes it so that the OpenVPN server will
not respond to packets unless those packets have a valid signature from a
pre-shared key. This makes it so that someone doing a port scan of your public
IP address will not see that the OpenVPN port is accepting connections. It also
provides other benefits described on the hardening page.

Setting this up is pretty simple. First you need to generate the pre-shared
key.

{% highlight console %}  
$ openvpn --genkey --secret ta.key  
{% endhighlight %}

Next, copy the key to `/config/auth` on the ERL and to all client machines.
Then set up the OpenVPN server to use the `--tls-auth` option.  

{% highlight console %}  
$ configure  
# set interfaces openvpn vtun0 openvpn-option "--tls-auth /config/auth/openvpn/ta.key 0"  
# commit  
# save  
{% endhighlight %}  

Finally, configure clients to pass the `--tls-auth ta.key 1` option to OpenVPN.

_**Update 2016-12-30:**_

Since writing this post I've employed a few addtional hardening options for 
OpenVPN:

- Drop root privileges after OpenVPN initialization. This is done by passing the 
  `--user nobody --group nogroup` options to OpenVPN. Additionally the 
  `--persist-key --persist-tun` options should be used to avoid the need for 
  privileges on soft restart.
- Use AES256 for the cipher and SHA256 for the message digest instead of the 
  defaults (Blowfish/SHA1). Note that this may impact performance.

Use the following commands to enable these options.

{% highlight console %}  
$ configure  
# edit interfaces openvpn vtun0
# set openvpn-option "--user nodbody"
# set openvpn-option "--group nogroup"
# set openvpn-option --persist-key
# set openvpn-option --persist-tun
# set encryption aes256
# set hash sha256
# commit  
# save  
{% endhighlight %}

You will also need to set the cipher and message digest appropriately in your 
client configuration.

# Conclusion

In this post we've covered one fairly common scenario for setting up an OpenVPN
server on the ERL. OpenVPN is incredibly flexible though, so if your needs
aren't completely covered by this guide there's a pretty good chance that you
just need to tweak the configuration.

The next post will wrap up this series by covering various useful
[odds and ends]({% post_url 2016-03-22-ubiquiti-edgerouter-lite-setup-part-6-odds-and-ends %})
that we haven't touched on yet.
