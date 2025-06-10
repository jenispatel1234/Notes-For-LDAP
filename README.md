# Notes-For-LDAP
LDAP information 
🐧 Zoo IT - Domain Login Prototype (LDAP Server + Workstation Setup)
✅ LDAP Server Setup (core-auth.zoo.local)
🔹 Step 1: Install Required Packages
bash
Copy
Edit
sudo apt update
sudo apt install slapd ldap-utils -y
sudo dpkg-reconfigure slapd
During configuration:

Omit OpenLDAP configuration? → No

DNS domain name → zoo.local

Organization name → Zoo

Admin password → zoo123 (or your own — must remember)

Remove DB when purging slapd? → No

Move old database? → Yes

🔹 Step 2: Create Base LDAP Directory Structure
Create base.ldif:

ldif
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
🔹 Step 3: Add Users Manually (Optional for testing)
Create users.ldif:

ldif
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
userPassword: {SSHA}PUT_HASHED_PASSWORD_HERE
loginShell: /bin/bash
homeDirectory: /home/ayusuf
Generate the password hash:

bash
Copy
Edit
slappasswd
Add the user:

bash
Copy
Edit
ldapadd -x -D "cn=admin,dc=zoo,dc=local" -W -f users.ldif
Verify:

bash
Copy
Edit
ldapsearch -x -b "dc=zoo,dc=local"
🖥️ LDAP Client (Workstation) Setup
🔹 Step 1: Install Required Packages
bash
Copy
Edit
sudo apt update
sudo apt install libnss-ldap libpam-ldap ldap-utils nscd -y
During prompts:

LDAP server URI: ldap://core-auth.zoo.local

Base DN: dc=zoo,dc=local

🔹 Step 2: Configure NSS
Edit:

bash
Copy
Edit
sudo nano /etc/nsswitch.conf
Update:

makefile
Copy
Edit
passwd:         files ldap
group:          files ldap
shadow:         files ldap
🔹 Step 3: Configure PAM (Home Directory Creation)
Edit PAM session:

bash
Copy
Edit
sudo nano /etc/pam.d/common-session
Add to the bottom:

bash
Copy
Edit
session required pam_mkhomedir.so skel=/etc/skel umask=0022
Enable via:

bash
Copy
Edit
sudo pam-auth-update
✔ Check: Create home directory on login

🔹 Step 4: Restart Services
bash
Copy
Edit
sudo systemctl restart nscd
🔹 Step 5: Test LDAP Login
Try:

bash
Copy
Edit
su - ayusuf
ldapsearch -x -H ldap://core-auth.zoo.local -b "dc=zoo,dc=local"
⚙️ Automation Scripts for User Management
🟢 activate.sh – Add a New LDAP User
bash
Copy
Edit
#!/bin/bash

BASE_DN="dc=zoo,dc=local"
OU="ou=People,$BASE_DN"
GID=10001

echo "Full Name,Username,Department,Birthday,Age,Pronouns: "
read -r LINE
IFS=',' read -r FULLNAME USERNAME DEPT BDAY AGE PRONOUNS <<< "$LINE"

read -s -p "Create password for $USERNAME: " PASSWORD
echo
HASH=$(slappasswd -s "$PASSWORD")
UID=$((RANDOM + 10000))

GIVENNAME="${FULLNAME%% *}"
SURNAME="${FULLNAME##* }"

cat <<EOF > /tmp/$USERNAME.ldif
dn: uid=$USERNAME,$OU
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: $USERNAME
sn: $SURNAME
givenName: $GIVENNAME
cn: $FULLNAME
displayName: $FULLNAME
uidNumber: $UID
gidNumber: $GID
userPassword: $HASH
loginShell: /bin/bash
homeDirectory: /home/$USERNAME
description: Department: $DEPT | Birthday: $BDAY | Age: $AGE | Pronouns: $PRONOUNS
EOF

ldapadd -x -D "cn=admin,$BASE_DN" -W -f /tmp/$USERNAME.ldif && echo "✅ User $USERNAME added!"
🔁 reset.sh – Reset a User’s Password
bash
Copy
Edit
#!/bin/bash

BASE_DN="dc=zoo,dc=local"
OU="ou=People,$BASE_DN"

read -p "User to reset password for: " USERNAME
read -s -p "Enter new password: " PASSWORD
echo
HASH=$(slappasswd -s "$PASSWORD")

cat <<EOF > /tmp/reset_$USERNAME.ldif
dn: uid=$USERNAME,$OU
changetype: modify
replace: userPassword
userPassword: $HASH
EOF

ldapmodify -x -D "cn=admin,$BASE_DN" -W -f /tmp/reset_$USERNAME.ldif && echo "🔁 Password reset for $USERNAME!"
❌ terminate.sh – Remove a User Permanently
bash
Copy
Edit
#!/bin/bash

BASE_DN="dc=zoo,dc=local"
OU="ou=People,$BASE_DN"

read -p "User to terminate: " USERNAME

ldapdelete -x -D "cn=admin,$BASE_DN" -W "uid=$USERNAME,$OU" && echo "❌ User $USERNAME permanently terminated!"
📝 Tips
Always back up your LDIF files before making changes.

You can view entries using:

bash
Copy
Edit
ldapsearch -x -LLL -b "ou=People,dc=zoo,dc=local"
Store sensitive scripts securely (consider file permissions or encryption).

