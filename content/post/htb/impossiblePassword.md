---
title: "ImpossiblePassword"
date: 2020-03-22T23:32:51-04:00
description: "patching a vulnerable binary with pwntools"
draft: false
---
```python
#!/usr/bin/env python3
# hackthebox.eu challenge: "impossible password"
# author: rmccarth@andrew.cmu.edu

from pwn import *
import os

#turn off pwntools elf logging
context.log_level = 'critical'

# load our ELF
elf = ELF("impossible_password.bin")

# the location of the jz instruction that we want to overwrite with a jnz instruction (75)
# 00000960: c7e8 cafc ffff 85c0 750c 488d 45c0 4889  ........u.H.E.H.)

#choosing 750c since it is unique in the binary
target_address = elf.search(b"\x75\x0c")
#retrieve the address of the hex that we want to change
target_address = next(target_address)

#make the change, substituting in our jnz instruction
elf.write(target_address, b"\x74\x0c")
#save the patched binary as run4flag
elf.save("./run4flag")

#make it executable, echo the first line to bypass the initial check, and then format for the flag
os.system("chmod +x run4flag && echo 'SuperSeKretKey' | ./run4flag | grep 'HTB' | cut -d '*
' -f 3")
```