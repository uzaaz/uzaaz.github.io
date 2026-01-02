---
layout: single
header:
  overlay_image: /assets/images/banner-012.jpg
  overlay_filter: 0.6
toc: true
toc_depth: 6
toc_label: Table of Contents
last_modified_at: 2026-01-02T10:53:00+01:00
categories:
  - Identity-and-Access-Management
  - Authentication
  - Homelab
tags:
  - OpenLDAP
  - Nextcloud
  - Centralized-Authentication
  - Directory-Services
  - CentOS
  - LDAP
  - IAM
  - SSO
---

In this lab, I integrated Nextcloud with an OpenLDAP directory on CentOS Stream 9 to provide a single source of truth for user identities and to validate application logins through LDAP binds and directory searches.​

**Security Benefits:**

- Reduces local account management and improves consistency by centralizing identities in LDAP.​
- Helps enforce stronger authentication practices by protecting LDAP simple binds with encryption (TLS/LDAPS) to avoid cleartext credentials on the network.​
- Improves auditability and troubleshooting by relying on deterministic LDAP filters and attributes during authentication.​

In this guide, I’ll share how I deployed an OpenLDAP server, created a basic directory structure and test users, then connected Nextcloud using the official “LDAP user and group backend” integration. 
After following these steps, my Nextcloud instance authenticates users directly against LDAP using a predictable Base DN and a reliable login filter (Nextcloud supports the **%uid** placeholder for matching the typed username).​

**What I’ll cover:**

- Installing and configuring OpenLDAP on CentOS Stream 9 and building a minimal directory tree (dc/ou).
- Creating LDAP users with SSHA password hashes and validating binds using `ldapwhoami` / `ldapsearch`.
- Enabling Nextcloud’s LDAP integration and configuring Base DN, login attributes, and user filters (including `%uid`).​
- Connectivity validation and common troubleshooting patterns before moving from CLI to GUI.​
- Hardening notes: why LDAP binds should be protected with TLS/LDAPS in real environments.​

---
## **Lab Environment:**

- **OS:** CentOS Stream 9 (VMware VMs)
- **Directory Service:** OpenLDAP (slapd)
- **Application:** Nextcloud + “LDAP user and group backend” integration​
- **Directory Model:** `dc=example,dc=com` with `ou=users`
- **Login Attribute:** `uid` (with optional email mapping via `mail`)

---

## Part 1: OpenLDAP Server Setup (VM1)

### 1. Install OpenLDAP
```bash
sudo dnf install epel-release -y
sudo dnf --enablerepo=epel install -y openldap-servers openldap-clients
```

### 2. Start and Enable LDAP Service
```bash
sudo systemctl enable slapd
sudo systemctl start slapd
sudo systemctl status slapd
```

### 3. Configure Firewall
```bash
sudo firewall-cmd --add-service=ldap --permanent
sudo firewall-cmd --reload
```

### 4. Generate Admin Password Hash
```bash
slappasswd
```
Save both the plaintext password and the generated hash (format: `{SSHA}xxxxx...`).

### 5. Set Admin Password
```bash
cat > adminpw.ldif << 'EOF'
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}YOUR_HASH_HERE
EOF
```
Replace `{SSHA}YOUR_HASH_HERE` with your actual hash, then apply:
```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f adminpw.ldif
```

### 6. Load Required Schemas
```bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/core.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

### 7. Configure Domain and Root DN
```bash
cat > chdomain.ldif << 'EOF'
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=example,dc=com

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=admin,dc=example,dc=com

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootPW
olcRootPW: {SSHA}YOUR_HASH_HERE
EOF
```
Replace the hash and apply:
```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f chdomain.ldif
```

Verify configuration:
```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=config" "(olcDatabase={2}mdb)" olcSuffix olcRootDN -LLL
```

### 8. Create Directory Structure
```bash
cat > base.ldif << 'EOF'
dn: dc=example,dc=com
objectClass: top
objectClass: domain
dc: example

dn: cn=admin,dc=example,dc=com
objectClass: organizationalRole
cn: admin

dn: ou=users,dc=example,dc=com
objectClass: organizationalUnit
ou: users
EOF
```

Apply the structure (enter your plaintext password when prompted):
```bash
sudo ldapadd -x -D "cn=admin,dc=example,dc=com" -W -f base.ldif
```

### 9. Test Admin Authentication
```bash
ldapwhoami -x -D "cn=admin,dc=example,dc=com" -W
```
Should return: `dn:cn=admin,dc=example,dc=com`

### 10. Create Test User
Generate password hash:
```bash
slappasswd
```

Create user file:
```bash
cat > bob.ldif << 'EOF'
dn: uid=bob,ou=users,dc=example,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
cn: Bob Smith
sn: Smith
uid: bob
userPassword: {SSHA}YOUR_HASH_HERE
mail: bob@example.com
EOF
```

Add user:
```bash
sudo ldapadd -x -D "cn=admin,dc=example,dc=com" -W -f bob.ldif
```

### 11. Verify User Creation
```bash
ldapsearch -x -LLL -b "ou=users,dc=example,dc=com" uid=bob
```

Test user authentication:
```bash
ldapwhoami -x -D "uid=bob,ou=users,dc=example,dc=com" -W
```

### 12. Get Server IP
```bash
ip addr show | grep "inet " | grep -v 127.0.0.1
```
Note the IP address (e.g., 192.168.x.x).

---

## Part 2: Nextcloud Server Setup (VM2)

### 1. Install PHP LDAP Module
```bash
sudo dnf install openldap-clients -y
sudo dnf install php-ldap -y
```

### 2. Restart Web Services
```bash
sudo systemctl restart php-fpm
sudo systemctl restart httpd
```

### 3. Verify PHP Module
```bash
php -m | grep ldap
```

### 4. Test Connectivity to LDAP Server
```bash
ping -c 3 192.168.x.x
ldapsearch -x -H ldap://192.168.x.x -D "cn=admin,dc=example,dc=com" -W -b "dc=example,dc=com" "(objectClass=inetOrgPerson)"
```
This should return Bob's user entry.

### 5. Enable LDAP App in Nextcloud
1. Login to Nextcloud as admin
2. Navigate to **Apps** → **Integration**
3. Find "**LDAP user and group backend**"
4. Click **Enable**

### 6. Configure LDAP Connection
Go to **Settings** → **LDAP/AD Integration**

**Server Tab:**
- **Host:** `ldap://192.168.x.x`
- **Port:** `389`
- **User DN:** `cn=admin,dc=example,dc=com`
- **Password:** Your admin plaintext password
- **Base DN:** `dc=example,dc=com` *(NOT ou=users)*

Click **Test Base DN** - should show success.

**Users Tab:**
- Click **↓ objectClass**
- Set: `(objectClass=inetOrgPerson)`

**Login Attributes Tab:**
- **LDAP/AD Username:** `uid`
- **LDAP/AD Email Address:** `mail`
- **Display Name:** `cn`

### 7. Test User Login
Logout from Nextcloud admin account and login with:
- **Username:** `bob`
- **Password:** Bob's plaintext password

User should authenticate successfully without being manually created in Nextcloud.

---
## Security Hardening

### Enable TLS/SSL Encryption
Unencrypted LDAP transmits credentials in cleartext. Configure LDAPS (LDAP over TLS):

#### 1) Generate (or obtain) certificates
```bash
sudo mkdir -p /etc/openldap/certs

# Self-signed lab certificate (replace CN with your LDAP server FQDN or IP) 
sudo openssl req -new -x509 -nodes \   
-out /etc/openldap/certs/ldapserver.pem \  
-keyout /etc/openldap/certs/ldapserver.key \  
-days 365

# Set permissions
sudo chown -R ldap:ldap /etc/openldap/certs
sudo chmod 600 /etc/openldap/certs/ldapserver.key
```

#### 2) Configure slapd to use TLS (cn=config)
Create an LDIF to set the TLS certificate paths in the dynamic config.
```sh
cat > tls.ldif << 'EOF'
dn: cn=config
changetype: modify
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/ldapserver.pem
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/ldapserver.key
-
replace: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/openldap/certs/ldapserver.pem
EOF
```

Apply it:
```sh
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f tls.ldif
```

#### 3) Restart slapd and open LDAPS port (optional but recommended)
```sh
sudo systemctl restart slapd

# If you want LDAPS on 636 (in addition to StartTLS on 389)
sudo firewall-cmd --add-service=ldaps --permanent
sudo firewall-cmd --reload
```

#### 4) Quick validation tests
```sh
# StartTLS on 389
ldapsearch -x -H ldap://192.168.x.x -ZZ -b "dc=example,dc=com" "(uid=bob)"

# LDAPS on 636 (if enabled)
ldapsearch -x -H ldaps://192.168.x.x:636 -b "dc=example,dc=com" "(uid=bob)"
```

**Note:** In a real environment, use a CA-signed certificate and distribute the CA certificate to clients so they can validate the LDAP server identity.


---
## Troubleshooting

### Connection Issues
Check LDAP service and firewall:
```bash
sudo systemctl status slapd
sudo firewall-cmd --list-services
sudo netstat -tulpn | grep 389
```

### Invalid Credentials Error
Verify domain configuration:
```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=config" "(olcDatabase={2}mdb)" olcSuffix olcRootDN olcRootPW -LLL
```

### User Not Found
Search directory:
```bash
ldapsearch -x -LLL -b "dc=example,dc=com" "(uid=bob)"
```

### Nextcloud "Could not connect to LDAP"
- Ensure Base DN is `dc=example,dc=com` (not `ou=users,dc=example,dc=com`)
- Use the correct LDAP filter: `(&(objectClass=inetOrgPerson)(uid=*))`
- Test from command line first to verify connectivity

---
## Quick Reference

### Add New User
```bash
slappasswd
nano newuser.ldif
sudo ldapadd -x -D "cn=admin,dc=example,dc=com" -W -f newuser.ldif
```

### Search All Users
```bash
ldapsearch -x -LLL -b "ou=users,dc=example,dc=com"
```

### Delete User
```bash
ldapdelete -x -D "cn=admin,dc=example,dc=com" -W "uid=username,ou=users,dc=example,dc=com"
```

### Restart LDAP Service
```bash
sudo systemctl restart slapd
```

---
## Key Points

1. **Always use `dc=example,dc=com` as Base DN in Nextcloud** - not `ou=users,dc=example,dc=com`
2. **Password vs Hash:** Use plaintext password for authentication, use hash in LDIF files
3. **Domain consistency:** All domain components must match across all configuration files
4. **Correct LDAP filter:** `(&(objectClass=inetOrgPerson)(uid=*))` works reliably in Nextcloud
5. **Test from command line first** before configuring Nextcloud web interface

---

## Reset OpenLDAP (if needed)

To start completely fresh:
```bash
sudo systemctl stop slapd
sudo rm -rf /var/lib/ldap/*
sudo rm -rf /etc/openldap/slapd.d/*
sudo dnf reinstall openldap-servers -y
sudo systemctl start slapd
```

---

**Successfully configured centralized authentication with OpenLDAP and Nextcloud!**

---

## Key Learnings & Best Practices

Through this implementation, I discovered several important considerations:​

**1. The importance of a clean LDAP design**  
Carefully planning the directory structure (`dc`, `ou`, and `cn` objects) makes integration with applications like Nextcloud much more reliable and easier to troubleshoot.​

**2. Clear separation between plaintext passwords and hashes**  
Using plaintext passwords only for authentication and SSHA hashes only inside LDIF files reduces mistakes, improves security, and avoids configuration confusion during bind operations.​

**3. Always validate from the CLI before the GUI**  
Testing LDAP connectivity and queries with `ldapsearch`, `ldapwhoami`, and `ping` before configuring the Nextcloud web interface helps quickly isolate whether issues are on the directory side or the application side.​

**4. Consistent Base DN and filters are critical**  
Using a consistent Base DN (`dc=example,dc=com`) and a robust user filter like `(&(objectClass=inetOrgPerson)(uid=*))` prevents “user not found” and “could not connect to LDAP” errors and ensures that only valid user objects are exposed to Nextcloud.​

---

## Credits & References

**Original Lab Exercise:**  
TP2: Mise en place d’une solution de la gestion d’identité pour le cloud computing  
Pr. Samia EL H. - Sécurité du Cloud Computing

**Implementation & Documentation:**  
Tested and verified on CentOS 9 Stream (January 2025)

**References:**
- [OpenLDAP Official Documentation](https://www.openldap.org/) - Core LDAP server implementation and administration guide​
- [Nextcloud LDAP/AD Integration Documentation](https://docs.nextcloud.com/server/stable/admin_manual/configuration_user/user_auth_ldap.html) - Official integration configuration reference​
- [LDAP Admin Tool](https://www.ldapadmin.org/) - GUI-based LDAP directory management and browser utility for testing and administration
- [LDAP Authentication Management Best Practices](https://identitymanagementinstitute.org/ldap-authentication-management-best-practices/) - Security controls and monitoring recommendations​