🧠 LDAP Notes (Easy & Short)
For Zoo IT System – Domain Login Prototype
🔧 Server: core-auth.zoo.local
💻 Client: Zoo Staff Workstation

🖥️ LDAP Server Setup (core-auth.zoo.local)
✅ Step 1: Install LDAP packages
bash
Copy
Edit
sudo apt update  
sudo apt install slapd ldap-utils -y  
sudo dpkg-reconfigure slapd
📝 During setup:

Domain: zoo.local

Org name: Zoo

Admin password: zoo123

✅ Step 2: Create base folders in LDAP
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
Add it to LDAP:

bash
Copy
Edit
ldapadd -x -D "cn=admin,dc=zoo,dc=local" -W -f base.ldif
✅ Step 3: Add users to LDAP
Create user file:

bash
Copy
Edit
nano users.ldif
Add user info (change UID, name). Use this:

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
userPassword: {SSHA}PASTE_HASH_HERE
loginShell: /bin/bash
homeDirectory: /home/ayusuf
Create password hash:

bash
Copy
Edit
slappasswd
Paste that hash in the .ldif file

Add user:

bash
Copy
Edit
ldapadd -x -D "cn=admin,dc=zoo,dc=local" -W -f users.ldif
✅ Step 4: Test LDAP entries
bash
Copy
Edit
ldapsearch -x -b "dc=zoo,dc=local"
💻 LDAP Workstation Setup (Zoo Staff)
✅ Step 1: Install LDAP client packages
bash
Copy
Edit
sudo apt update  
sudo apt install libnss-ldap libpam-ldap ldap-utils nscd -y
Set:

LDAP server: ldap://core-auth.zoo.local

Base DN: dc=zoo,dc=local

✅ Step 2: Connect login to LDAP
Edit:

bash
Copy
Edit
sudo nano /etc/nsswitch.conf
Update these lines:

makefile
Copy
Edit
passwd: files ldap  
group:  files ldap  
shadow: files ldap
✅ Step 3: Create user home on login
bash
Copy
Edit
sudo nano /etc/pam.d/common-session
Add at bottom:

bash
Copy
Edit
session required pam_mkhomedir.so skel=/etc/skel umask=0022
Then run:

bash
Copy
Edit
sudo pam-auth-update
✔️ Tick: "Create home directory"

✅ Step 4: Restart service
bash
Copy
Edit
sudo systemctl restart nscd
✅ Step 5: Test user login
bash
Copy
Edit
su - ayusuf
⚙️ Automation Scripts (Optional Bonus)
activate.sh → Creates a new user

reset.sh → Resets a user's password

terminate.sh → Deletes a user permanently
