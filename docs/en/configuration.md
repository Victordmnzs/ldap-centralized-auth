# Step-by-step configuration

[← Back to README](../../README.en.md) · [Português](../pt-br/configuracao.md)

Base DN used across the whole infrastructure: `dc=aluno,dc=seg,dc=ctism,dc=ufsm,dc=br`

---

## 1. LDAP server (Ubuntu)

### 1.1 Packages

OpenLDAP, the command-line tools and the LDAP Account Manager (LAM) web interface:

```bash
sudo apt install slapd ldap-utils ldap-account-manager
```

### 1.2 Base domain

```bash
sudo dpkg-reconfigure slapd
```

The wizard was given the DNS domain (`aluno.seg.ctism.ufsm.br`), which produces the Base DN above, the organization name and the administrator (`cn=admin`) password.

### 1.3 Directory tree (DIT) through LAM

The tree was built through the LAM web interface, accessed locally on the server:

1. Configured the `ou=People` and `ou=group` OU suffixes.
2. Created the `gerencia` and `usuarios` groups.
3. Created the users, both with `posixAccount` and `inetOrgPerson`:
   - `victor`: uidNumber 10000, group `gerencia`
   - `usuario`: uidNumber 10001, group `usuarios`

### 1.4 `stats` log level

To make `slapd` log every operation (CONNECT, SEARCH, BIND and their results), the log level was changed with the LDIF [`server/log_stats.ldif`](../../server/log_stats.ldif):

```ldif
dn: cn=config
changetype: modify
replace: olcLogLevel
olcLogLevel: stats
```

```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f log_stats.ldif
```

`-Y EXTERNAL -H ldapi:///` authenticates over `slapd`'s local socket using the system root identity, which is how `cn=config` is modified.

---

## 2. Linux client (Ubuntu)

### 2.1 Integration packages

```bash
sudo apt install libnss-ldapd libpam-ldapd ldap-utils
```

### 2.2 NSLCD

`nslcd` translates NSS and PAM requests into LDAP queries. The wizard was given the LDAP server IP and the Base DN:

```bash
sudo dpkg-reconfigure nslcd
```

### 2.3 NSS

Enables the `passwd`, `group` and `shadow` databases over LDAP in `/etc/nsswitch.conf`:

```bash
sudo dpkg-reconfigure libnss-ldapd
```

![libnss-ldapd configuration](../img/libnss-ldapd.png)

### 2.4 PAM: automatic home directory

Line added to `/etc/pam.d/common-session` ([`linux-client/common-session.snippet`](../../linux-client/common-session.snippet)):

```
session required pam_mkhomedir.so skel=/etc/skel umask=0022
```

### 2.5 Privileges for the `gerencia` group

Line added to sudoers with `sudo visudo` ([`linux-client/sudoers.snippet`](../../linux-client/sudoers.snippet)):

```
%gerencia ALL=(ALL:ALL) ALL
```

The `%` marks `gerencia` as a group, resolved from LDAP through NSS.

### 2.6 Tests

- **`victor` (gerencia):** successful LDAP login and `sudo tcpdump -i any -c 5` allowed.
- **`usuario` (usuarios):** successful LDAP login, with `/home/usuario` created automatically by PAM. `sudo tcpdump -i any -c 5` was blocked and the incident was logged.

The logs for these tests are in the [log analysis](log-analysis.md#2-linux-client--varlogauthlog).

---

## 3. Windows 10 Pro client (extra credit)

### 3.1 Environment

The Windows 10 Pro VM ran locally on VirtualBox, connected to the institution network over VPN to reach the OpenLDAP server. Without Active Directory, Windows cannot authenticate against LDAP natively. So **pGina** was installed: an open-source Credential Provider that replaces the native logon component.

### 3.2 pGina LDAP plugin

The LDAP plugin was enabled on the **Plugin Selection** tab with these settings:

| Setting | Value |
|---|---|
| LDAP server | IP of the Ubuntu server running `slapd` |
| Port | 389 |
| Base DN | `dc=aluno,dc=seg,dc=ctism,dc=ufsm,dc=br` |
| DN lookup mode | *Search for DN* enabled, filter `(uid=%u)` |

With this, pGina first runs an **anonymous search** for the entry whose `uid` matches the name typed at the logon screen and, once it finds the full DN, performs the **BIND** with the password provided.

### 3.3 Validation by simulation

The test used pGina's **Simulation** tab, which emulates the full authentication process without touching the active session. The *Emulate pGina Service* method was used with `usuario`'s credentials.

![pGina simulation](../img/pgina-simulacao.png)

| Stage | Plugin | Result | Meaning |
|---|---|---|---|
| Authentication | LDAP | True | Entry found in the directory and credentials validated |
| Authentication | Local Machine | False | Expected: there is no local account with that name, so authentication was resolved entirely by LDAP |
| Authorization | LDAP | True | pGina's *Always allow* policy |
| Gateway | LDAP / Local Machine | True | Session setup steps completed; on a real login Windows would create the local profile |

The final result (✅ next to *Result*) confirms every stage completed.
