# Write-up
This is a mock/brief write up of the challenge.

## Summary
I was able to identify a few critical vulnerabilities in the web page and the host machine that ulimately allowed root access. Weak security controls, poor patch management and proper account permissions are recommended to resolve these issues.

## Attack narrative
Whilst there are a couple of generic "prompts" available for path guidance for this machine, I chose to approach it as a black-box - using nothing but the IP as a starting point.

## Findings

### Joomla CMS SQLi vulnerability
- Outdated Joomla version is still in use and is vulnerable to injection.

### Steps to reproduce 
- Run a script such as one found here https://github.com/stefanlucas/Exploit-Joomla/blob/master/joomblah.py from the terminal or attempt manual SQLi via input fields.

### Expected result
- No sensetive information should be returned on malicious entry input.

### Actual result
- The app returns a list of registered users and their password hashes.

### Recommendations
- Update to the latest version of Joomala CMS.
- Enforce input validation/ sanitization to prevent generic SQLi
- Ensure hashes are at least salted to discourage basic hash brute force.

---

### php shell upload
- By accessing the `templates` menu I was able to upload the code for a reverse php shell, gaining access to the host.

### Steps to reproduce
- From the Dashboard, go to `Configuration` , `Templates` and `Protostar`. 
- Enter code for your desired shell and save.
- Set up a listener on your machine.
- Navigate to `MACHINE_IP/templates/protostar/SHEEL_FILE_NAME`
- Check listener for your shell.


### Expected result
- No malicious shell code should be able to be uploaded.

### Actual result
- Reverse shell can be placed in the templates folder.

### Recommendations
- Update to the latest version of Joomala CMS.
- Reduce the number of accounts that can upload code to the host and ensure they are properly secured.
- Check/ modify .htaccess file to configure who can access writeable files.
- Implement code validation / authorization on sensetive, writable files to ensure they can not be edited without approval.

---

### User account access via clear text password storage
- By viewing a php.config file I was able to gain a password for a user account.

### Steps to reproduce
- Read or navigate to the applications php.config file to view user passwords.

### Expected result
- Clear text passwords should not be visable.

### Actual result
- Able to view user passwords

### Recommendations
- Restrict read/ access permissions to .config files
- Store passwords securely - salted hashes, restricted access 

---

### Weak permissions allowed root access
- The user account was able to be elvated to root via executing `yum` as root.

### Steps to reproduce
- Once logged into the user account, spawn a root shell via yum : 

``` 
TF=$(mktemp -d)
cat >$TF/x<<EOF
[main]
plugins=1
pluginpath=$TF
pluginconfpath=$TF
EOF

cat >$TF/y.conf<<EOF
[main]
enabled=1
EOF

cat >$TF/y.py<<EOF
import os
import yum
from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
requires_api_version='2.1'
def init_hook(conduit):
  os.execl('/bin/sh','/bin/sh')
EOF

sudo yum -c $TF/x --enableplugin=y 
``` 

### Expected result
- Privesc should not be possible via running the yum command

### Actual result
- The command spawns a root shell.

### Recommendations
- Check account privileges and who can run which commands as `sudo`. Follow the principle of Least Privilage.

--- 

### General Recommendations
- Hide/ forbid sensitive pages such as /administrator from being discoverable/ accessible from the public facing website to prevent unwanted access and/or tampering.


---

# Walkthrough
This is a more step by step of the actual process of hacking the machine.
Technologies/tools used:
- nmap
- gobuster
- JohnTheRipper
- linpeas


## Enumeration
Starting out with a basic scan we can see there are a couple of ports open.
![nmap](https://user-images.githubusercontent.com/103790652/218285114-f4b6987c-977e-4069-a4fe-c0d36757781e.png)

The webpage served is pretty basic but a gobuster scan reveals more about the site (notable pages: admin + multiple others that suggest possible file upload).

![gobuster](https://user-images.githubusercontent.com/103790652/218285545-78f84b84-78b8-4681-9bf4-2fe41f5f99fe.png)


Running a quick vlunrability scan against the site returns an interesting result - the site uses Joomla! CMS and is vulnerable to SQLi!

![vuln result](https://user-images.githubusercontent.com/103790652/218285261-cada9046-c786-4a7c-932f-7328a8112efc.png)

## Exploitation

Running the exploit against the page yields a few usernames, as well as a bcrypt hash.

![joom](https://user-images.githubusercontent.com/103790652/218285303-8f4b9622-a418-4c18-a0aa-91c69d3053f0.png)

Feeding the hash to JohnTheRipper yeilds the great password of ############ after a wait.

Tried to log in via ssh using the obtained usernames + password but had no luck so returned to the webpage scan results.

Found a log in page at /administrator and tried the credentials again -- success.

Upon log in, greeted by generic Joomala dashboard.

Tried to upload shell file via the media folder but there seemed to be some sort of client side filtering prventing the file being delivered/ server side hiding of any uploaded files so I decided to check elsewhere on the site for easier access and return later.

After some searching, found that files could be added/written within the website template folder and successfully uploaded a reverse php shell.

![shell](https://user-images.githubusercontent.com/103790652/218285526-d0671de3-2257-46bd-928c-f1851e9a6602.png)

Navigating to the shell's URL triggers it and a local listener is able to catch the response.

![path](https://user-images.githubusercontent.com/103790652/218285613-44560eff-7cdc-41d4-9292-4b8ebdd587e4.png)
![catch](https://user-images.githubusercontent.com/103790652/218285615-231dce4d-4fbe-4f0e-93a5-7400bb196362.png)

Achieved a stable shell via experimenting with the "magic" route found here as I hadn't used it before:

https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/

Now moved to /tmp as Read/Write permissions were present and uploaded linpeas.sh (https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS) via local machine web server.

Linpeas showed 2 possible routes of privilege escalation: 

A clear text password:

![pass](https://user-images.githubusercontent.com/103790652/218285845-5e3ee25c-65e9-4112-a154-f8ecae9abb24.png)

and direct root access via CVE-2021-4034:

![cve](https://user-images.githubusercontent.com/103790652/218285862-1969f37d-9dde-46eb-9605-bc7909a72aeb.png)

## Password / account privesc

Very simply, this just involved switching to the user jjameson (found in /home) with the password found by linpeas. 

![switchuser](https://user-images.githubusercontent.com/103790652/218285981-b737f694-059e-4d87-ad43-3f57345e66fe.png)

Once the account was switched, the user.txt flag was readable from the Desktop.

Root privesc in this path was done first by finding out what I could get away with (run):

![sudo -l](https://user-images.githubusercontent.com/103790652/218285995-45da6dd1-5498-4726-98ca-e7612ded7f16.png)

Checking GTFObins it was apparent that yum can spawn interactive root shell by loading a custom plugin when it doesn't drop sudo rights

![gtfobins](https://user-images.githubusercontent.com/103790652/218286023-b2ea3946-d28d-441c-8360-567673fb6f2a.png)

Running/ loading the plugin was enough to get root privesc and allowed access to the root.txt flag.

![jj-root](https://user-images.githubusercontent.com/103790652/218286071-37b3ea09-359c-450a-b7a8-47999aef412e.png)


## CVE exploitation
After obtaining root via the above method, I returned to the CVE out of curiousity. Supposedly the version of Polkit (policy manager) could be exploited to allow any user to run commands as root.
I decided to check out polkit on the machine & found that polkit's SUID was set and the version was indeed vlunerable.

![suid](https://user-images.githubusercontent.com/103790652/218286302-bee9aebb-0550-4198-82b2-748351be29d4.png)

![version](https://user-images.githubusercontent.com/103790652/218286305-9dab0ee0-d0de-4d45-8828-810023104d77.png)

After spending some time researching, I found an exploit that would work perfectly at https://packetstormsecurity.com/files/165739/PolicyKit-1-0.105-31-Privilege-Escalation.html. 

I crafted the 3 necessary files on my local machine and downloaded them to the vulnerable machine via webserver.
After compiling the code, the exploit was formed.

![files](https://user-images.githubusercontent.com/103790652/218286421-ee1b6cb1-2d74-40b4-9dae-3a879f2efb83.png)

I simply changed the file permissions to allow the exploit to run and let it work it's magic.

![permissions](https://user-images.githubusercontent.com/103790652/218286464-fdca9dd2-c9e3-4982-9c84-13cc258768c8.png)

Instant root! 

![root](https://user-images.githubusercontent.com/103790652/218286475-d6f9094d-7684-49f1-95df-2c5bd6af83ee.png)


Overall, I found this machine to be good fun, enjoyed looking into the exploit (given it's severity) and liked the fact there were multiple ways to exploit the target - I learned a lot.



















