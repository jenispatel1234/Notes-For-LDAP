# Notes-For-LDAP
LDAP information 
##### Server Creation Process
@ Step 1: Install Packages
- sudo apt update
- sudo apt install slapd ldap-utils -y
- sudo dpkg-reconfigure slapd
      - terminal2 (for zoolab assignment) OpenLDAP server configuration? --> **No**
      - Domain name: **zoo.local**
      - Organization name: **Zoo**
      - Admin password: **zoo123**    (can be anything)    *must remember*
      - Remove database when slapd is purged? --> **No**
      - Move old database? --> **Yes**
@ Step 2: Create Base Directory Tree
- nano base.ldif
      dn: ou=People,dc=zoo,dc=local
      objectClass: organizationalUnit
      ou: People
      dn: ou=Groups,dc=zoo,dc=local
      objectClass: organizationalUnit
      ou: Groups
- ldapadd -x -D "cn=admin,dc=zoo,dc=local" -W -f base.ldif
2 Step 3: Create Users
- nano users.ldif
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
- slappasswd
      - enter a password, and you will get a decrypted SSHA password you need to put in the {SSHA}PUT_HASHED_PASSWORD_HERE section in order to work.
- ldapadd -x -D "cn=admin,dc=zoo,dc=local" -W -f users.ldif
- Verify the user is created: ldapsearch -x -b "dc=zoo,dc=local"

##### Workstation Creation Process
@ Step 1: Install Packages
- sudo apt update
- sudo apt install libnss-ldap libpam-ldap ldap-utils nscd -y
      - LDAP server URL: ldap://core-auth.zoo.local
      - DN of search base: dc=zoo,dc=local
@ Step 2: Configure NSS and PAM to Use LDAP
- sudo nano /etc/nsswitch.conf
- Update to following:
      - passwd:         files ldap
      - group:          files ldap
      - shadow:         files ldap
@ Step 3: Configure PAM Authentication
- sudo nano /etc/pam.d/common-session
- Add at the very bottom:
      - session required pam_mkhomedir.so skel=/etc/skel umask=0022
- pam-auth-update
      - check create home directory on login by pressing space and enter
@ Step 4: Restart
- systemctl restart nscd
@ Step 5: Test LDAP Authentication
- su - ayusuf
- ldapsearch -x -H ldap://core-auth.zoo.local -b "dc=zoo,dc=local"

##### Create Automation Scripts
@ Step 1: activate.sh - activates a user
      #!/bin/bash  
        
      BASE_DN="dc=zoo,dc=local"  
      ou="ou=People,$BASE_DN"  
      GID=10001  
        
      echo "Full Name,Username,Department,Birthday,Age,Pronouns: "  
      read -r LINE  
        
      IFS=',' read -r FULLNAME USERNAME DEPARTMENT BIRTHDAY AGE PRONOUNS <<< "$LINE"  
        
      read -s -p "Create a password for $USERNAME: " PASSWORD  
      echo  
      HASH=$(slappasswd -s "$PASSWORD")  
      uid=$((RANDOM + 10000))  
        
      GIVENNAME="${FULLNAME%% *}"  
      SURNAME="{FULLNAME##* }"  
        
      cat <<EOF > /tmp/$USERNAME.ldif  
      dn: uid=$USERNAME,ou=People,dc=zoo,dc=local  
      objectClass: inetOrgPerson  
      objectClass: posixAccount  
      objectClass: shadowAccount  
      uid: $USERNAME  
      sn: $SURNAME  
      givenName: $GIVENNAME  
      cn: $FULLNAME  
      displayName: $FULLNAME  
      uidNumber: $uid  
      gidNumber: $GID  
      userPassword: $HASH  
      loginShell: /bin/bash  
      homeDirectory: /home/$USERNAME  
      description: Department: DEPARTMENT | Birthday: BIRTHDAY | Age: AGE | Pronouns: PRONOUNS  
      EOF  
        
      ldapadd -x -D "cn=admin,$BASE_DN" -W -f /tmp/$USERNAME.ldif && echo "User $USERNAME added!"

@ Step 2: reset.sh - resets a user password
      #!/bin/bash  
        
      BASE_DN="dc=zoo,dc=local"  
      OU="ou=People,$BASE_DN"  
        
      read -p "User password to reset: " USERNAME  
        
      read -s -p "Enter new password: " PASSWORD  
      echo  
      HASH=$(slappasswd -s "$PASSWORD")  
        
      cat <<EOF > /tmp/reset_$USERNAME.ldif  
      dn: uid=$USERNAME,$OU  
      changetype: modify  
      replace: userPassword  
      userPassword: $HASH  
      EOF  
        
      ldapmodify -x -D "cn=admin,$BASE_DN" -W -f /tmp/reset_$USERNAME.ldif && echo "Password reset for $USERNAME!"

@ Step 3: terminate.sh - terminates a user permanently
      #!/bin/bash  
        
      BASE_DN="dc=zoo,dc=local"  
      OU="ou=People,$BASE_DN"  
        
      read -p "User to terminate: " USERNAME  
        
      ldapdelete -x -D "cn=admin,$BASE_DN" -W "uid=$USERNAME,$OU" && echo "User $USERNAME permanently terminated!"
