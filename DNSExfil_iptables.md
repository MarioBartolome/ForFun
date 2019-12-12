When attempting to run `iptables` without SuperCow privileges on Debian 10 Buster, a SEGFAULT is produced. 

I found this little problem while (by error) tried to set a new rule on my OUTPUT chain without sudo: 
![image](https://user-images.githubusercontent.com/23175380/70739024-368e4180-1d16-11ea-85f7-4e1bbc39c44a.png)


Funny thing is... it took it a while to return. So it seems `iptables` tried to resolve, even though I did not have the permission to run such a task in the first place.

So, a (new?) path for DNS Exfiltration just appeared!

![image](https://user-images.githubusercontent.com/23175380/70739058-49a11180-1d16-11ea-9b09-47206d7564cd.png)


This may prove useful on some restricted environments where tools `ping`, `dig`, `nslookup`... etc are gone.

Enjoy!
