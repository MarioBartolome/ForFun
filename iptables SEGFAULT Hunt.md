So, as shown at a SEGFAULT is triggered. Let's have a look to it!

---

A quick look on `valgrind`, shows that iptables trying to pass a  NULL Pointer to a function to be used: 

![image](https://user-images.githubusercontent.com/23175380/70739376-f7142500-1d16-11ea-934a-090124d0cdf1.png)


So `gdb` was fired up, symbols and sources loaded into path and breakpoints set to those functions

![image](https://user-images.githubusercontent.com/23175380/70739417-1b700180-1d17-11ea-9d4d-f14121333dde.png)

## Tracing the SEGFAULT

The first breakpoint is hit, the code is about to call `xtables_main(...)`
![image](https://user-images.githubusercontent.com/23175380/70739443-2d51a480-1d17-11ea-88c7-98af56a0ccac.png)

---
It enters the function and some of the code is displayed, but not all of it... I took the time to review the code to see what it is actually doing: 

![image](https://user-images.githubusercontent.com/23175380/70739465-393d6680-1d17-11ea-95bc-c61c417caa4b.png)

A more handy “trace” of how far we are right now:

![image](https://user-images.githubusercontent.com/23175380/70739503-49554600-1d17-11ea-8968-e858a5cef0be.png)
---

Continue and hit the next breakpoint, where some of the parameters passed to the function do_commandx are easily displayed:

![image](https://user-images.githubusercontent.com/23175380/70739526-540fdb00-1d17-11ea-911b-eefc8b3c5926.png)
![image](https://user-images.githubusercontent.com/23175380/70739541-5e31d980-1d17-11ea-9278-517c7f951d80.png)

---

Continue to the next function, `add_entry`, shows some code that might seem complicated at first sight, but nevertheless: 
![image](https://user-images.githubusercontent.com/23175380/70739607-802b5c00-1d17-11ea-95e3-0e9242373844.png)

Once reviewed, the interesting code for this function is shown below: 

![image](https://user-images.githubusercontent.com/23175380/70739619-89b4c400-1d17-11ea-8e12-2ab22e005c02.png)

---

The next function, `nft_rule_insert`  is an interesting one, as here is where the last working pieces of code are ran. 
Keep an eye on `rulenum` variable, as this is the var that will cause an uninitalized pointer to be passed to the next function.

![image](https://user-images.githubusercontent.com/23175380/70739664-a5b86580-1d17-11ea-8d8e-20bdb59b124d.png)

Here, a pointer to a `nftnl_rule_list` struct called `list` is created, but never initialized due to the `rulenum` being 0

![image](https://user-images.githubusercontent.com/23175380/70739700-b7017200-1d17-11ea-9cfd-32a9481dbc55.png)

![image](https://user-images.githubusercontent.com/23175380/70739730-c8e31500-1d17-11ea-8a3c-a4a22eee6e00.png)

Therefore, a var called `new_rule` is initialized at line 2139, but the function that fills it, `nft_rule_add`,  returns `NULL` due to the lack of privileges: 

![image](https://user-images.githubusercontent.com/23175380/70739763-de583f00-1d17-11ea-85b4-656344d02a73.png)

---

And the last function of our breakpointed list arrives: 

![image](https://user-images.githubusercontent.com/23175380/70739781-ed3ef180-1d17-11ea-9b5f-713fcbcd1f1d.png)


As shown, `list_add` is called on line 782, trying to dereference a NULL pointer, by accessing its `list` parameter.

![image](https://user-images.githubusercontent.com/23175380/70739805-f9c34a00-1d17-11ea-84ec-626e3a5d8c44.png)



And that's it. SEGFAULT you!

