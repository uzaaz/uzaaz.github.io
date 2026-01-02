---
layout: single
title: "SSH Two-Factor Authentication (2FA) Setup Guide - CentOS 9 Stream"
header:
  overlay_image: /assets/images/banner-012.jpg
  overlay_filter: 0.6
toc: true
toc_depth: 6
toc_label: Table of Contents
last_modified_at: 2025-12-31T10:27:00+01:00
categories:
  - Security
  - Linux
  - System Administration
tags:
  - SSH
  - 2FA
  - Google Authenticator
  - CentOS 9
  - PAM
  - TOTP
  - Authentication
  - Security Hardening
---

Two-factor authentication (2FA) is a critical security control that adds an extra layer of protection beyond just passwords. In this lab, I implemented 2FA for SSH access on CentOS 9 Stream using Google Authenticator's TOTP (Time-based One-Time Password) implementation.

**Security Benefits:**
- Mitigates password-based attacks (brute force, credential stuffing)
- Provides defense-in-depth against compromised credentials
- Meets compliance requirements for multi-factor authentication

In this guide, I'll share how I implemented two-factor authentication (2FA) for SSH on CentOS 9 Stream using Google Authenticator. After following these steps, my SSH access now requires both a password and a time-based one-time password (TOTP) from my smartphone, which significantly improved security against unauthorized access attempts.

**What I'll cover:**
- How I installed and configured the Google Authenticator PAM module
- Setting up TOTP codes with mobile app integration
- Configuring the SSH daemon for challenge-response authentication
- The SELinux permissions fixes I discovered for stable 2FA operation
- Emergency access procedures and setting up 2FA for multiple users

---
**Lab Environment:**
- **OS:** CentOS 9 Stream
- **Authentication Method:** TOTP (RFC 6238)
- **PAM Module:** google-authenticator-libpam
- **Mobile App:** Google Authenticator

---
## 2FA Installation

### Step 1: Install Google Authenticator
```bash
sudo dnf install epel-release -y
sudo dnf install google-authenticator qrencode-libs -y
```

**What this does:** Installs the PAM module and QR code support.

---
### Step 2: Verify PAM Module Installed
```bash
ls -la /usr/lib64/security/pam_google_authenticator.so
```

**Expected:** File exists with proper permissions.

---
### Step 3: Switch to Your User Account
```bash
su - user
```

**Important:** Replace `user` with your actual username. Do NOT run as root unless you want 2FA for root.

---
### Step 4: Generate Secret Key
```bash
google-authenticator
```

---
### Step 5: Answer All Questions with 'y'

- Time-based tokens? **y**
- Update file? **y**
- Disallow multiple uses? **y**
- Time skew? **y**
- Rate limiting? **y**

**📱 SCAN THE QR CODE** with Google Authenticator app on your phone NOW!

**📝 WRITE DOWN THE EMERGENCY CODES** and store them safely!

---
### Step 6: Exit Back to Root
```bash
exit
```

---
### Step 7: Verify Configuration File Created
```bash
ls -la /home/user/.google_authenticator
```

**Expected:** File exists, owned by `user:user`.

---
### Step 8: Fix Permissions
```bash
sudo chown user:user /home/user/.google_authenticator
sudo chmod 600 /home/user/.google_authenticator
```

**Replace `user` with your username.**

---
### Step 9: Set SELinux to Permissive Mode
```bash
sudo setenforce 0
```

---
### Step 10: Make SELinux Permissive Permanent
```bash
sudo nano /etc/selinux/config
```

Change:
```
SELINUX=enforcing
```

To:
```
SELINUX=permissive
```

Save and exit.

**What this does:** Prevents SELinux from blocking 2FA file access.

---
### Step 11: Fix SELinux Context
```bash
sudo restorecon -v /home/user/.google_authenticator
```

**Replace `user` with your username.**

---
### Step 12: Verify File Permissions and Context
```bash
ls -la /home/user/.google_authenticator
ls -Z /home/user/.google_authenticator
```

**Expected:**
- Permissions: `-rw-------`
- Owner: `user:user`
- SELinux context: Should NOT show `?`

---
### Step 13: Configure PAM for SSH
```bash
sudo nano /etc/pam.d/sshd
```

Find the line:
```
auth       substack     password-auth
```

Add this line **immediately AFTER it**:
```
auth       required     pam_google_authenticator.so no_increment_hotp
```

**Result should look like:**
```
#%PAM-1.0
auth       substack     password-auth
auth       include      postlogin
auth       required     pam_google_authenticator.so no_increment_hotp
```

**CRITICAL:** The Google Authenticator line MUST be AFTER `password-auth`.

Save and exit.

---
### Step 14: Verify PAM Configuration
```bash
sudo cat /etc/pam.d/sshd | head -10
```

Confirm Google Authenticator line is AFTER `password-auth`.

---
### Step 15: Configure SSH Daemon
```bash
sudo nano /etc/ssh/sshd_config
```

Add or modify these lines:
```
ChallengeResponseAuthentication yes
KbdInteractiveAuthentication yes
UsePAM yes
```

Save and exit.

---
### Step 16: Verify SSH Configuration
```bash
sudo sshd -t
```

**Expected:** No output (means configuration is valid).
```bash
sudo grep -E "Challenge|KbdInteractive|UsePAM" /etc/ssh/sshd_config
```

**Expected:** All three set to `yes`.

---
### Step 17: Restart SSH Service
```bash
sudo systemctl restart sshd
sudo systemctl status sshd
```

**Expected:** `Active: active (running)` in green.

---
### Step 18: Test 2FA Login

**⚠️ KEEP YOUR CURRENT SSH SESSION OPEN!**

From Windows or another terminal:
```bash
ssh user@YOUR_IP_HERE
```

**Expected flow:**

1. **Password prompt:** Enter your regular password
2. **Verification code prompt:** Enter 6-digit code from Google Authenticator app
3. **Success:** You're logged in

**⏱️ Note:** Code changes every 30 seconds - enter it quickly!

---
## Verification Commands

After successful setup, verify everything:
```bash
# Check package installed
rpm -qa | grep google-authenticator

# Verify PAM module
ls -la /usr/lib64/security/pam_google_authenticator.so

# Check user config
ls -la /home/user/.google_authenticator

# Verify SSH config
sudo sshd -T | grep -E "challengeresponse|kbdinteractive|usepam"

# Check SSH status
sudo systemctl status sshd

# Check SELinux mode
getenforce
```

**Expected SELinux mode:** `Permissive`

---
## Emergency Access

### Using Scratch Codes
If you lose your phone, use one of the emergency scratch codes instead of the 6-digit code when prompted for "Verification code".

**Note:** Each scratch code works only once.

---
### Disable 2FA via Console

If locked out and have console/physical access:
```bash
sudo nano /etc/pam.d/sshd
```

Comment out:
```
#auth       required     pam_google_authenticator.so no_increment_hotp
```
```bash
sudo systemctl restart sshd
```

---
## Enable 2FA for Additional Users

Repeat steps 3-12 for each user:
```bash
# Switch to user
su - username

# Run setup
google-authenticator

# Answer y to all questions
exit

# Fix permissions (as root)
sudo chown username:username /home/username/.google_authenticator
sudo chmod 600 /home/username/.google_authenticator
sudo restorecon -v /home/username/.google_authenticator
```

---
## Time Synchronization

TOTP codes are time-based. Ensure server time is accurate:
```bash
# Enable NTP
sudo timedatectl set-ntp true

# Start chronyd
sudo systemctl enable --now chronyd

# Verify time
date
```

---
## Clean Uninstall - Start Fresh

If you previously attempted 2FA setup, clean everything first:

### Remove PAM Configuration
```bash
sudo nano /etc/pam.d/sshd
```

Comment out or delete any line containing `google_authenticator`:
```
#auth       required     pam_google_authenticator.so
```

Save and exit.

---
### Reset SSH Configuration
```bash
sudo nano /etc/ssh/sshd_config
```

Set these to `no`:
```
ChallengeResponseAuthentication no
KbdInteractiveAuthentication no
```

Save and exit.

---
### Restart SSH
```bash
sudo systemctl restart sshd
```

---
### Test Password-Only Login
```bash
ssh user@your_server_ip
```

Should work with password only, no verification code.

---
### Remove Old Configuration Files
```bash
sudo rm /home/user/.google_authenticator
sudo rm /root/.google_authenticator 2>/dev/null
```

---
### Verify Clean State
```bash
# Check PAM config is clean
sudo grep google /etc/pam.d/sshd

# Check SSH config
sudo grep -E "Challenge|KbdInteractive" /etc/ssh/sshd_config

# Verify no config files exist
ls -la /home/user/.google_authenticator
```

All should return empty or "No such file or directory".

---
## Key Learnings & Best Practices

Through this implementation, I discovered several important considerations:

**1. PAM Configuration Order Matters**
The Google Authenticator module must be placed AFTER `password-auth` in `/etc/pam.d/sshd`. Incorrect ordering causes authentication failures where the system requests the verification code before validating the password.

**2. SELinux Context Management**
SELinux can block PAM module access to user configuration files. Setting SELinux to permissive mode resolves this for lab environments, but production systems should use proper SELinux policies (`no_increment_hotp` option helps minimize file write requirements).

**3. Always Maintain Backup Access**
- Keep one active SSH session open during configuration
- Save emergency scratch codes in a secure location

**4. Time Synchronization is Critical**
TOTP codes are time-sensitive (30-second windows). NTP synchronization between server and mobile device is essential for reliable operation.

---
## Credits & References

**Original Lab Exercise:**  
TP2: Mise en place d'une solution d'authentification multi-facteur  
Pr. Samia EL H. - Sécurité du Cloud Computing

**Implementation & Documentation:**  
Tested and verified on CentOS 9 Stream (December 2025)

**References:**
- [Google Authenticator PAM Module](https://github.com/google/google-authenticator-libpam)
- [RFC 6238 - TOTP: Time-Based One-Time Password Algorithm](https://tools.ietf.org/html/rfc6238)
- [CentOS SSH Security Documentation](https://docs.centos.org/)
- [Linux PAM Documentation](http://www.linux-pam.org/)
