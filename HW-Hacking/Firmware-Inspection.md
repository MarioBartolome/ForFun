This is the second of three mini-posts where I will write about some fun discoveries

- 1/3: [Dissecting a ~~piñata~~Router](https://github.com/MarioBartolome/ForFun/blob/master/HW-Hacking/Dissecting-a-router.md)
- **2/3: CVE-2019-14919, CVE-2019-14920. Inspecting a router's firmware and reversing binaries to achieve RCE**
- 3/3: [CVE-2019-14918. Stored XSS via DHCP injection](https://github.com/MarioBartolome/ForFun/blob/master/HW-Hacking/XSS-Injection-via-DHCP-requests.md)

# 0x02/0x03 : Firmware inspection and binary reversing to achieve RCE

## Seeking some interesting info

While still intact, I ran an Nmap scanner against the router. It found an open telnet service running behind port 2301 (naughty naughty), which presents a login when opening a session to it.

![Telnet logon at port 2301](https://user-images.githubusercontent.com/23175380/64639364-fca45b00-d407-11e9-9757-a10ae58412bb.png)

But telnet has no login support by default... so it may be a modified version of telnet, or the system is using the ```-l``` flag on telnet command to set a binary that will take care of the login.

During the firmware inspection, and by having a look to the file ```rcS``` contained at ```etc_ro```, the login management software behind the scenes can be found:

![Login management sw behind the telnet service](https://user-images.githubusercontent.com/23175380/64637833-e943c080-d404-11e9-93a9-1bcf5df2766c.png)


So let's reverse it!


#### Reversing the login management binary
--------

- CVE-2019-14919. Affected product/version: Billion Smart Energy Router SG600R2. Firmw v3.02.rc6


This is kinda embarrassing... Just by checking the strings at the binary, the "Login incorrect" message appears to be next to the **correct username and password**. Straight away in clear-text. Embedded into the source. Some awesome programming practices here. 

Let's go step by step. Start by instructing **radare2** to analyze the binary by issuing `aaaa` to perform an in-deep analysis. 

Then, by having a look to the strings at the data sections (use `iz` in radare2 to get information about the strings in data sections), the *Login incorrect* address seems to be stored in the `.rodata` section at address `0x00400d44`.

![Strings found at binary](https://user-images.githubusercontent.com/23175380/64925410-677ed900-d7f0-11e9-932c-4f727f887a7c.png)

By checking the cross-references the string has (use `axt 0x00400d44`to `a`nalyze `x`refs `t`o address), it's kinda easy to find that it was used only at the login process:

![XRefs to string](https://user-images.githubusercontent.com/23175380/64925414-7c5b6c80-d7f0-11e9-9430-8de64abb8b0e.png)

So by navigating to that address (`s 0x400ad8` to `s`eek to an specific address) and checking the graphic view (`VV` to change to graphic View), the whole login process comes clear: 

![HardCoded credentials](https://user-images.githubusercontent.com/23175380/64637855-f8c30980-d404-11e9-90bd-fdf67860bdd8.png)

And yup, the login works 

![Root shell](https://user-images.githubusercontent.com/23175380/64637883-05dff880-d405-11e9-8aef-4b1c998a183c.png)


#### "Hidden" Root WebShell
--------

- CVE-2019-14920. Affected product/version: Billion Smart Energy Router SG600R2. Firmw v3.02.rc6

When rains, it pours! 

While checking the file system, a really eye-catching name came up: 

![...the fuck is that?](https://user-images.githubusercontent.com/23175380/64637929-15f7d800-d405-11e9-92db-94c33db9087a.png)

The other files are listed as part of the administration of the router, but not this one... there's no sign of it at the panels:

![No system command here](https://user-images.githubusercontent.com/23175380/64638236-ae8e5800-d405-11e9-804b-ebfa85a02830.png)

Feels like Christmas... Let's open it!

A hidden root webshell. All for me...

![A root WebShell all for you!](https://user-images.githubusercontent.com/23175380/64638289-c239be80-d405-11e9-8ede-33c3eba3ed64.png)


### And last but not least

Oh, yeah, the admin password... I found a file where the username and the password are stored: 

![Super hard password](https://user-images.githubusercontent.com/23175380/64638351-d41b6180-d405-11e9-9044-ea67e4cdc37c.png)


Also a couple of wild stored XSS appeared... One of them with a really interesting exploitation path. 

Check on the next part of this mini-writeup to find out more! 

---

If you readed the whole ~~annoying~~ post you may want to know the user/pass: 

user: `hsinchu-binos2` \\\\ pass: `33659498`

