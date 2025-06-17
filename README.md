LDAP Notes – Zoo IT Login System (Easy & Clear)
This project is about making one central login system for the Zoo staff. One main LDAP Server keeps all the users, and the staff workstation connects to it to login using same ID.

 Part 1: LDAP Server Setup (core-auth.zoo.local)
 Step 1: Install LDAP server packages
We start by installing OpenLDAP on the server:

bash
Copy
Edit
sudo apt update  
sudo apt install slapd ldap-utils -y  
sudo dpkg-reconfigure slapd
 While setting up, give:

Domain Name → zoo.local

Organisation Name → Zoo

Password for admin → zoo123

Keep database → Yes

Move old DB → Yes

This makes the base LDAP server ready.

Step 2: Create base directory (People & Groups)
We now create folders in LDAP for users and groups:

bash
Copy
Edit
nano base.ldif
Paste this:

makefile
Copy
Edit
dn: ou=People,dc=zoo,dc=local
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=zoo,dc=local
objectClass: organizationalUnit
ou: Groups
Then run:

bash
Copy
Edit
ldapadd -x -D "cn=admin,dc=zoo,dc=local" -W -f base.ldif

This makes “People” and “Groups” folders inside LDAP.

Step 3: Add User (e.g., ayusuf)
We add a real user to LDAP.

First make file:

bash
Copy
Edit
nano users.ldif
Paste:

makefile
Copy
Edit
dn: uid=ayusuf,ou=People,dc=zoo,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: ayusuf
sn: Yusuf
cn: Amina Yusuf
uidNumber: 10001
gidNumber: 10001
userPassword: {SSHA}PASTE_HERE
loginShell: /bin/bash
homeDirectory: /home/ayusuf
Now get password hash:

bash
Copy
Edit
slappasswd
Enter a password (like 12345) and it will give you encrypted password like {SSHA}...
Paste that in the file.

Add user:

bash
Copy
Edit
ldapadd -x -D "cn=admin,dc=zoo,dc=local" -W -f users.ldif
Now your user is added to the server.

Step 4: Check if user exists
bash
Copy
Edit
ldapsearch -x -b "dc=zoo,dc=local"
This shows all users in the LDAP server. You should see ayusuf.

Part 2: LDAP Client Setup (Zoo Staff Workstation)
This machine is used by staff. We make it connect to LDAP server.

Step 1: Install LDAP client tools
bash
Copy
Edit
sudo apt update  
sudo apt install libnss-ldap libpam-ldap ldap-utils nscd -y
During install:

LDAP server: ldap://core-auth.zoo.local

Base DN: dc=zoo,dc=local

Version: 3

Step 2: Connect login system to LDAP
Edit this file:

bash
Copy
Edit
sudo nano /etc/nsswitch.conf
Change these lines:

makefile
Copy
Edit
passwd: files ldap  
group:  files ldap  
shadow: files ldap
This tells the system to check users from LDAP also.

Step 3: Make home folder on login
Edit:

bash
Copy
Edit
sudo nano /etc/pam.d/common-session
At the bottom, add:

bash
Copy
Edit
session required pam_mkhomedir.so skel=/etc/skel umask=0022
Then run:

bash
Copy
Edit
sudo pam-auth-update
Tick "Create home directory on login"

Step 4: Restart name service
bash
Copy
Edit
sudo systemctl restart nscd

Step 5: Test login
Try:

bash
Copy
Edit
su - ayusuf
If it logs in and creates a folder /home/ayusuf, everything is working 🎉

Bonus: Automation Scripts
activate.sh
This script helps you create a new user fast.
It asks for full name, username, etc., and then makes .ldif and adds user to LDAP.

reset.sh
This script is for resetting password of a user if they forget it.
You type username and new password, and it updates LDAP.

terminate.sh
This script deletes a user forever from LDAP.
Just type the username, and it removes them.

Summary (in your words)
I learned how to make a central login system using LDAP

LDAP server stores all user info

Client machines connect and allow login from server

This helps in big networks like Zoo or companies
