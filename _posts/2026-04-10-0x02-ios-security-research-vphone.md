---
title: "0x02: Getting started in iOS Security Research using vphone-cli - Solving Challenges"
date: 2026-04-10
categories: [ios-security]
tags: [blog,mobile,ios,apple,security,research,lldb,radare2,vphone-cli]
---

For this second post about getting started in iOS security research, we will use the already configured environment to solve some arm64 exploitation challenges.

## Playing around with the virtualized iPhone

We will use our virtualized iPhone to solve some simple arm64 exploitation exercises extracted from the 8ksec.io blog posts.

### Before getting started...

Before starting with the challenges, we need to do some additional work in order to get our binaries to work on the virtualized iPhone. We will use the binary `vuln` provided by 8ksec's in their ARM64 Reversing And Exploitation series, used in Part 1 and 2.

#### Signing the binary

If we directly copy the `vuln` binary into the virtualized phone (having previously created the `challenges` folder):
```
scp -P 2222 vuln mobile@127.0.0.1:/var/mobile/challenges
```
And try to execute it, we will be prompted with the following error:
```
iPhone:/var/mobile/challenges root# ./vuln 
zsh: killed     ./vuln
```

This is because iOS requires a code signature on every Mach-O, even on our jailbroken vphone. We can use the `ldid` tool to sign it directly because in this jailbroken variant it won't be checked.
```
ldid -S vuln
```
If we now execute the signed version, we will be able to execute it normally:
```
iPhone:/var/mobile/challenges root# ./vuln
Better luck next time
```

#### Creating symlinks for the expected tools

There is another issue that I found while playing around with the exploitation exercises: the `vuln` binary isn't able to call `whoami`.

The output looks like this:
```
    iPhone:/var/mobile/challenges root# ./vuln heap helloworld.txt
    Address of main function is 0x1004dbb24
    Heap overflow challenge. Execute a shell command of your choice on the device
    Welcome: from helloworld.txt, printing out the current user
```
When it should look like this:
```
    iPhone:/var/mobile/challenges root# ./vuln heap helloworld.txt
    Address of main function is 0x1004dbb24
    Heap overflow challenge. Execute a shell command of your choice on the device
    Welcome: from helloworld.txt, printing out the current user
>>> root
```

If we investigate a little bit further, we will see that the whoami binary is located in `/var/jb/usr/bin/`, and this path is configured in our PATH environment variable so it should be accessible, but for some reason it is not for `vuln`. 
```
iPhone:/var/mobile/challenges root# whoami
root

iPhone:/var/mobile/challenges root# which whoami
/var/jb/usr/bin/whoami

iPhone:/var/mobile/challenges root# echo $PATH 
/usr/local/sbin:/var/jb/usr/local/sbin:/usr/local/bin:/var/jb/usr/local/bin:/usr/sbin:/var/jb/usr/sbin:/usr/bin:/var/jb/usr/bin:/sbin:/var/jb/sbin:/bin:/var/jb/bin:/usr/bin/X11:/var/jb/usr/bin/X11:/usr/games:/var/jb/usr/games
```

To address this issue, I simply decided to create a bunch of softlinks for those expected binaries so that they would be accessible from `/bin`. Doing this we faced another limitation, the filesystem is read-only:
```
iPhone:/var/mobile/challenges root# ln -s /var/jb/usr/bin/whoami /bin/whoami
ln: failed to create symbolic link '/bin/whoami': Read-only file system
```
Which we can simply solve by remount our root directory as writable:
```
iPhone:/var/mobile/challenges root# mount -u -w /
```

Creating the softlinks for our desired binaries didn't work at first, until we reached `bash`. Linking `/var/jb/usr/bin/bash` to `/bin/sh` solved the issue.
```
iPhone:/var/mobile/challenges root# ls /bin 
df  ps
iPhone:/var/mobile/challenges root# ./vuln heap helloworld.txt        
Address of main function is 0x104db3b24
Heap overflow challenge. Execute a shell command of your choice on the device
Welcome: from helloworld.txt, printing out the current user
iPhone:/var/mobile/challenges root# which bash                        
/var/jb/usr/bin/bash
iPhone:/var/mobile/challenges root# ln -s /var/jb/usr/bin/bash /bin/sh
iPhone:/var/mobile/challenges root# ls /bin                           
df  ps	sh
iPhone:/var/mobile/challenges root# ./vuln heap helloworld.txt        
Address of main function is 0x1004e3b24
Heap overflow challenge. Execute a shell command of your choice on the device
Welcome: from helloworld.txt, printing out the current user
root
```

Going a little bit further, if we reverse engineer the binary we will see that in reality, the binary calls `whoami` via the libc function `system`. From this function man page we can see that: _The system() function hands the argument command to the command interpreter sh(1)._ The conclusion I arrived at, is that `system` wasn't able to reach sh, and the lack of error handling in the binary was causing it to fail silently, which got fixed by adding the `/bin/sh` softlink.

Now we can finally move on with the challenges!

### Heap Overflow
Reference: [ARM64 Reversing And Exploitation Part 1 - ARM Instruction Set + Simple Heap Overflow](https://8ksec.io/arm64-reversing-and-exploitation-part-1-arm-instruction-set-simple-heap-overflow/)

_To be done..._

### Use After Free
Reference: [ARM64 Reversing and Exploitation Part 2 - Use After Free](https://8ksec.io/arm64-reversing-and-exploitation-part-2-use-after-free/)

_To be done..._

### ROP Chain
Reference: [ARM64 Reversing and Exploitation Part 3 - A Simple ROP Chain](https://8ksec.io/arm64-reversing-and-exploitation-part-3-a-simple-rop-chain/)

_To be done..._

