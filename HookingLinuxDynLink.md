# Hooking Linux's Dynamic Linker

As I was preparing a small workshop to teach some frida magic, I came across a problemo... The last demo prepared, had a bit of a complex nature. I was creating a demo following this scenario: 

- A main binary loads a Dynamic Library.
- The library was compiled without any symbols exported.
- The library had visibility hidden on compiler options.
- Once loaded, the Dynamic Linker will run the constructors defined on the library. 


As usual, I putted myself into more than I could digest at that moment...


## Constructors. A brief intro

When dealing with dynamically loaded libraries, sometimes it is needed to perform some actions before any symbol is resolved. We could think about this kind of behavior in a similar way as constructors do in Object Oriented languages. 

In fact, these are named on the very same way, and defined as follows:

```c
<RET_TYPE> __attribute__((constructor)) <FUNC_SIGNATURE>;
```


Those functions marked as "constructors" will be ran at Dynamic-loading time. As soon as `dlopen(...)` is called on the library. I.e: 


### testlib.c
```c
#include <stdio.h>

void before_main(void);
void main(void);

void __attribute__((constructor)) before_main(void);


void main() {
        printf("\tI am main\n");
}


void before_main(void) {
        printf("\tI am before_main\n");
}

```

To be compiled as `gcc testlib.c -fPIC -shared -o testlib.so`



### test.c
```c
#include <dlfcn.h>
#include <stdio.h>


int main() {
        void *h;
        void (*f_main)(void);


        printf("Dynamically linking library...\n");
        h = dlopen("./testlib.so", RTLD_LAZY);


        printf("Resolving symbol for main and calling it...\n");
        f_main =  dlsym(h, "main");
        (*f_main)();

        dlclose(h);
        return 0;
}
```

To be compiled as `gcc test.c -ldl -o testdynlink`

---

Once executed, it will display the text in the following order: 
```
Dynamically linking library...
        I am before_main
Resolving symbol for main...
        I am main
```


## Demo3

The demo follows the next structure: 



```c
#include <stdlib.h>                          
#include <stdio.h>                           
#include <dlfcn.h>                           
                                             
int main(int argc, char** argv){             
        printf("Hit ENTER to continue\n");
        getchar();

        dlopen("lib/libnative.so", RTLD_NOW);

        return 0;
}                                            
``` 
Saved as `demo3.c` and compiled as `gcc demo3.c -ldl -o demo3`


```c
#define MAX_FLAG_SIZE 12
char* getFlag();

```
Saved as `libnative.h`



```c
#include "libnative.h"
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <time.h>

char* getFlag(){
        srand(time(0));
        char *a = (char*) malloc(MAX_FLAG_SIZE + 1 * sizeof(char));
        for (int i = 0; i < MAX_FLAG_SIZE; ++i){
                a[i] = (char)( 'A' + (random() %26));
        }
        a[MAX_FLAG_SIZE] = '\0';
        return a;
}


void __attribute__((constructor)) main(void);

void main(void) {
    char *userResponse = (char*) malloc(MAX_FLAG_SIZE * sizeof(char));
    size_t size = MAX_FLAG_SIZE;
    
    printf("Ahoy! ARRRR! Gimme flag:\n");
    getline(&userResponse, &size, stdin);
    userResponse[strlen(userResponse) - 1] = '\0';
    
    if(strcmp(userResponse, getFlag()) == 0){
        printf("ARRRR! Where ye get my flag?!?!?!?!!!\n");
    } else {
        printf("That's not my flag!\n");
    }
}

```
Saved as `libnative.c`. And compiled as `gcc -shared -fPIC -o lib/libnative.so libnative.c -s -fvisibility=hidden`


## Hooking what?


The idea for the demo was to have a binary that will request a flag (it even talks like a pirate... Ahoy!). If the correct flag is provided, then happy face, sad face otherwise. 

**Simple.**

Simple, as long as you plan to hook `strcmp`. But that's boring.


Once you plan to exfiltrate the flag, by hooking the `getFlag` function available on `libnative.so`, or try to **actually** replace it by the one you entered, it becomes harder. 

There's no symbols, so no `findExportByName("libnative.so", "getFlag")` will help you.

There's no way to hook (at first glance) by offset, as once the library is loaded it will "load" the constructor as well, and that's where the whole logic of the program relies. 


The only chance is to hook Linux's Dynamic Linker itself. More precisely, hook where it begins to run the constructor, and then hook the library's `getFlag` function.


## Backtracing dlopen

Once [`dlopen(...)`](https://code.woboq.org/userspace/glibc/dlfcn/dlopen.c.html#87) is called, the Dynamic Linker calls [`dlopen_doit`](https://code.woboq.org/userspace/glibc/dlfcn/dlopen.c.html#57) which, at some point and via use of `_dl_catch_error`, makes use of [`dl_open_worker`](https://code.woboq.org/userspace/glibc/elf/dl-open.c.html#dl_open_worker) which will be calling [`_dl_init`](https://code.woboq.org/userspace/glibc/elf/dl-open.c.html#506) to start the constructors by calling our real target: [**`call_init`**](https://code.woboq.org/userspace/glibc/elf/dl-init.c.html#call_init)


If you want to follow the trace more in depth, I recommend reading the source available at: 

- [call_init](https://code.woboq.org/userspace/glibc/elf/dl-init.c.html#call_init)
- [_dl_init](https://code.woboq.org/userspace/glibc/elf/dl-init.c.html#_dl_init)
- [Macro DL_CALL_DT_INIT](https://code.woboq.org/userspace/glibc/sysdeps/generic/ldsodefs.h.html#85)
- [dl_open_worker](https://code.woboq.org/userspace/glibc/elf/dl-open.c.html#dl_open_worker)
- [dlopen_doit](https://code.woboq.org/userspace/glibc/dlfcn/dlopen.c.html#57)
- [dlopen](https://code.woboq.org/userspace/glibc/dlfcn/dlopen.c.html#dlopen)


All right! So just hooking `call_init` and then going for the kill by attempting to hook the `getFlag` function, by offset of course ;)


Unfortunately, that's not all folks. `call_init` is not exported as a symbol on `libdl-x`. So the function must be hooked by offset as well.

### Some gdb funk

Running the binary inside `gdb` and attempting to set a breakpoint at `call_init` will only succeed once the Dynamic Linker has come into play, so we'll need to run the binary at least once before attempting to break at `call_init`.

`gdb demo3`

Then let it run with `r`. Finish the execution and set a breakpoint on `call_init` with `b call_init`

![call_init breakpoint](https://user-images.githubusercontent.com/23175380/72224057-55caf980-3576-11ea-85c5-04521c1f8810.png)


Once done, run it again with `r` and continue (`c`) the execution until the loaded library comes up in the stack. Something like this should come up:

![A wild libnative.so appeared!](https://user-images.githubusercontent.com/23175380/72224094-b9edbd80-3576-11ea-8084-2d5a78ab7d74.png)


The backtrace can be of use as well, `bt`.

![Backtrace to call_init](https://user-images.githubusercontent.com/23175380/72224127-e6a1d500-3576-11ea-90b1-fa41610270a2.png)


At some point you will reach the desired absolute address. Then, by subtracting it the base address of `ld-x...` the offset for the `call_init` can be calculated. 

In my compilation (ld-2.23.so) it appears to be at an absolute address of `0x7ffff7de7630`. Checking the base address of `ld-x`is simple and there's many ways. I checked on the process maps via `/proc/<PID>/maps`:

![Maps for demo3 process](https://user-images.githubusercontent.com/23175380/72224143-24066280-3577-11ea-9fb1-24e1df8bd523.png)


So my base address for ld-2.23.so is `0x7ffff7dd7000`. Then `call_init_abs_address - ld_base_address = call_init_offset` or `0x7ffff7de7630 − 0x7ffff7dd7000 = 0x10630` 

That's the loving offset of `call_init`. 

![GDB on demo3](https://user-images.githubusercontent.com/23175380/72224149-397b8c80-3577-11ea-9c4a-5ecb0624e8aa.png)



## Frida funk!

Almost there... Then

```javascript
var ld_base_address = Module.findBaseAddress("ld-2.23.so");
var call_init_offset = 0x10630;
var call_init_abs = ld_base_address.add(call_init_offset);
Interceptor.attach(call_init_abs, {...});
```

Should do the trick to give us a chance to hook the `getFlag` function, which is also to be hooked by offset! 

I'll let you figure out the rest. It should not be hard to reverse the library and find the offset of the `getFlag` function.

![Frida!!!!](https://user-images.githubusercontent.com/23175380/72224256-ba875380-3578-11ea-97e8-22773dc80be5.png)




## References

- [GCC Visibility](https://gcc.gnu.org/wiki/Visibility)
- [Dynamic Library constructor and destructor attributes](https://www.apriorit.com/dev-blog/537-using-constructor-attribute-with-ld-preload)
- [Giving yourself a window to debug a shared library before DT_INIT - ANDROID!](http://www.giovanni-rocca.com/giving-yourself-a-window-to-debug-a-shared-library-before-dt_init-with-frida-on-android/)
- [Source code for the Dynamic Linker](https://code.woboq.org)
- [Frida snippets](https://github.com/iddoeldor/frida-snippets)
- [Frida API](https://frida.re/docs/javascript-api/)
- [PWNDbg](https://github.com/pwndbg/pwndbg)
- [Radare2](https://github.com/radareorg/radare2)


## Thanks to

- Ole André (@oleavr)
- Eduardo Novella (@enovella)
- Giovanni Rocca (@iGio90)


