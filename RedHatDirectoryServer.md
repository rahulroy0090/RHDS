# Red Hat Directory Server Installation and Configuration Guide

## Overview
This document outlines the complete installation and configuration process for Red Hat Directory Server (389-ds) on RHEL 9, including repository setup, package installation, instance creation, and LDAP data management.

## System Information
- **Hostname**: rhds.rbac.preprod.finopaymentbank.in
- **Operating System**: RHEL 9
- **IP Address**: x/24
- **Gateway**: x
- **Instance Name**: rbac
- **Base DN**: dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in

## Step 1: Repository Configuration

### Enable Required Repositories
```bash
# Enable base RHEL 9 repositories
subscription-manager repos --enable=rhel-9-for-x86_64-appstream-rpms --enable=rhel-9-for-x86_64-baseos-rpms

# Set release version
subscription-manager release --set=9

# Enable Directory Server repository
subscription-manager repos --enable=dirsrv-12.5-for-rhel-9-x86_64-rpms
```

### Verify Repository Configuration
```bash
# List enabled repositories
subscription-manager repos --list-enabled
```

**Expected Output:**
- rhel-9-for-x86_64-appstream-rpms (Enabled)
- rhel-9-for-x86_64-baseos-rpms (Enabled)
- dirsrv-12.5-for-rhel-9-x86_64-rpms (Enabled)

## Step 2: Package Installation

### Enable Directory Server Module
```bash
dnf module enable redhat-ds:12
```

### Install Directory Server Packages
```bash
# Clean package cache
dnf clean all
dnf clean packages
dnf makecache

# Install 389 Directory Server
dnf install 389-ds-base cockpit-389-ds
```

## Step 3: Directory Server Instance Creation

### Interactive Instance Creation
```bash
dscreate interactive
```

**Configuration Parameters:**
- **Hostname**: rhds.rbac.preprod.finopaymentbank.in
- **Instance Name**: rbac
- **Port**: 389 (LDAP)
- **Secure Port**: 636 (LDAPS)
- **Self-signed Certificate**: Yes
- **Directory Manager DN**: cn=Directory Manager
- **Database Backend**: bdb
- **Base Suffix**: dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
- **Sample Entries**: No
- **Top Suffix Entry**: Yes
- **Auto-start**: Yes

## Step 4: Service Verification

### Check Instance Status
```bash
# Check if instance is running
dsctl rbac status

# Check systemd service status
systemctl status dirsrv@rbac

# Verify listening ports
ss -tuln | grep -E '389|636'
```

**Expected Results:**
- Instance "rbac" is running
- Service is active (running)
- Ports 389 and 636 are listening

## Step 5: Backend Configuration

### Create Additional Backend
```bash
# Create new backend for simplified DN structure
dsconf slapd-rbac backend create --be-name userRoot2 --suffix "dc=finopaymentbank,dc=in"

# List all backends
dsconf slapd-rbac backend suffix list
```

**Available Backends:**
- dc=finopaymentbank,dc=in (userroot2)
- dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in (userroot)

## Step 6: LDAP Data Structure

### Organizational Units Structure
```ldif
# Users OU
dn: ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
objectClass: organizationalUnit
ou: Users

# Groups OU
dn: ou=Groups,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
objectClass: organizationalUnit
ou: Groups
```

### Group Definitions
```ldif
# Cluster Admin Group
dn: cn=cluster-admin,ou=Groups,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
objectClass: groupOfNames
cn: cluster-admin
member: uid=nitesh.jha,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in

# Read-Only Group
dn: cn=read-only,ou=Groups,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
objectClass: groupOfNames
cn: read-only
member: uid=jayesh.chavan,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in

# Admin Group
dn: cn=admin,ou=Groups,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
objectClass: groupOfNames
cn: admin
member: uid=komal.ambore,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
member: uid=akshay,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
member: uid=abhishek.tyagi,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
member: uid=mansi.rastogi,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
member: uid=rahul.kumar,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
member: uid=brijesh.yadav,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
member: uid=pankaj.gouni,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
member: uid=pravesh,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
member: uid=pawan.kumar,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
```

## Step 7: User Management

### User Entry Template
```ldif
dn: uid=username,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
objectClass: inetOrgPerson
uid: username
cn: Full Name
sn: Surname
```

### Add LDAP Data
```bash
# Add organizational structure and groups
ldapadd -x -D "cn=Directory Manager" -W -H ldap://localhost:389 -f rhds_users1.ldif

# Add user entries
ldapadd -x -D "cn=Directory Manager" -W -H ldap://localhost:389 -f rhds_clean_users.ldif
```

## Step 8: Password Management

### Generate SHA Password Hash
```bash
echo -n "password" | openssl dgst -sha1 -binary | openssl base64
```

### Password Modification Template
```ldif
dn: uid=username,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in
changetype: modify
replace: userPassword
userPassword: {SHA}PHZ8Qa+xKtoUAZDtgts/2TDi76M=
```

### Apply Password Changes
```bash
ldapmodify -x -D "cn=Directory Manager" -W -f userPassword1.ldif
```

## Step 9: Testing and Verification

### Test User Authentication
```bash
# Test user binding
ldapwhoami -x -D "uid=nitesh.jha,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in" -W

# Search as user
ldapsearch -x -D "uid=nitesh.jha,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in" -W -b "dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in"
```

### Basic LDAP Queries
```bash
# Search entire directory
ldapsearch -x -H ldap://localhost:389 -D "cn=Directory Manager" -W -b "dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in"

# Search specific user
ldapsearch -x -D "uid=username,ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in" -W -b "ou=Users,dc=rhds,dc=rbac,dc=preprod,dc=finopaymentbank,dc=in" "(uid=username)"
```

## Troubleshooting

### Common Issues and Solutions

1. **Invalid Credentials Error (49)**
   - Verify Directory Manager password
   - Check DN format accuracy
   - Ensure proper password hashing format

2. **Service Not Starting**
   - Check systemd service status: `systemctl status dirsrv@rbac`
   - Review logs: `journalctl -xeu dirsrv@rbac`

3. **Port Binding Issues**
   - Verify ports are not in use: `ss -tuln | grep -E '389|636'`
   - Check firewall settings
   - Verify SELinux context

## User Directory Structure

### Current Users in System
| Username | Full Name | Group Membership |
|----------|-----------|------------------|
| nitesh.jha | Nitesh Jha | cluster-admin |
| jayesh.chavan | Jayesh Chavan | read-only |
| komal.ambore | Komal Ambore | admin |
| akshay | Akshay | admin |
| abhishek.tyagi | Abhishek Tyagi | admin |
| mansi.rastogi | Mansi Rastogi | admin |
| rahul.kumar | Rahul Kumar | admin |
| brijesh.yadav | Brijesh Yadav | admin |
| pankaj.gouni | Pankaj Gouni | admin |
| pravesh | Pravesh | admin |
| pawan.kumar | Pawan Kumar | admin |

## Security Considerations

1. **Default Password**: All users currently have the same password hash ({SHA}PHZ8Qa+xKtoUAZDtgts/2TDi76M=)
2. **SSL/TLS**: Server configured with self-signed certificates on port 636
3. **Access Control**: Consider implementing proper ACIs (Access Control Instructions)

## Maintenance Commands

### Regular Maintenance
```bash
# Start/Stop/Restart instance
dsctl rbac start
dsctl rbac stop
dsctl rbac restart

# Backup directory
dsctl rbac db2ldif
dsctl rbac ldif2db

# Monitor logs
tail -f /var/log/dirsrv/slapd-rbac/access
tail -f /var/log/dirsrv/slapd-rbac/errors
```

## Next Steps

1. Configure proper Access Control Instructions (ACIs)
2. Set up SSL certificates from trusted CA
3. Implement password policies
4. Configure replication for high availability
5. Set up monitoring and alerting
6. Integrate with external authentication systems if required
