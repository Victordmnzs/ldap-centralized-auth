# Log analysis

[← Back to README](../../README.en.md) · [Português](../pt-br/analise-de-logs.md)

Full excerpts are in [`docs/logs/`](../logs/). Base DN, test user, IP and hostnames were sanitized.

---

## 1. LDAP server — `journalctl -u slapd`

```bash
sudo journalctl -u slapd -n 30 --no-pager
```

With logging at the `stats` level, `usuario`'s authentication from the Linux client can be followed in three phases.

### 1.1 SEARCH: find the user's DN

`nslcd` searches the whole tree (`scope=2`) for the entry with that `uid`. The first attempt used a mistyped uid:

```
conn=1007 op=6 SRCH base="dc=aluno,..." scope=2 filter="(&(objectClass=posixAccount)(uid=usuaro))"
conn=1007 op=6 SEARCH RESULT tag=101 err=0 nentries=0
```

Right after, with the correct uid:

```
conn=1007 op=7 SRCH base="dc=aluno,..." scope=2 filter="(&(objectClass=posixAccount)(uid=usuario))"
conn=1007 op=7 SEARCH RESULT tag=101 err=0 nentries=1
```

- `err=0`: the search operation itself had no error.
- `nentries=0` / `nentries=1`: no entry, or exactly one entry, found.

The server also answers precisely for non-existent users, exposing nothing beyond "no entry".

### 1.2 BIND: validate the password

With the DN found, the client opens a new connection (`conn=1017`) and binds:

```
conn=1017 fd=19 ACCEPT from IP=<IP_CLIENTE>:36614
conn=1017 op=0 BIND dn="cn=Usuario Teste,ou=People,dc=aluno,dc=seg,dc=ctism,dc=ufsm,dc=br" method=128
conn=1017 op=0 BIND dn="cn=Usuario Teste,..." mech=SIMPLE bind_ssf=0 ssf=0
conn=1017 op=0 RESULT tag=97 err=0 qtime=0.000022 etime=0.000268
```

- `method=128`: SIMPLE authentication (username and password).
- `err=0` on `RESULT tag=97`: the server validated the password.
- `ssf=0`: session **without** a security layer (no TLS), expected in an internal lab.
- `qtime` / `etime`: queue and execution times in the microsecond range.

### 1.3 Follow-up queries

After the BIND, `nslcd` runs further SEARCHes to fetch the account attributes (`shadowAccount`) PAM needs to finish the login. All of them return `err=0`.

---

## 2. Linux client — `/var/log/auth.log`

```bash
sudo tail -n 30 /var/log/auth.log
```

### 2.1 `victor` (gerencia): sudo allowed

```
sudo: victor : TTY=pts/1 ; PWD=/home/victor ; USER=root ; COMMAND=/usr/bin/tcpdump -i any -c 5
sudo: pam_unix(sudo:session): session opened for user root(uid=0)
sudo: pam_unix(sudo:session): session closed for user root
```

`victor` authenticated via LDAP and, as a member of the `gerencia` group mapped in sudoers, opened a root session to run `tcpdump`. Both the opening and the closing of the session are logged.

### 2.2 `usuario` (usuarios): home created automatically

```
su: (to usuario) ldapclient on pts/1
su: pam_unix(su-l:session): session opened for user usuario(uid=10001)
```

UID 10001, defined in the LDAP directory, is recognized by the local system. The terminal also showed `Creating directory '/home/usuario'`, printed by `pam_mkhomedir`.

### 2.3 `usuario` (usuarios): sudo denied

```
sudo: pam_unix(sudo:auth): authentication failure; logname=ldapclient uid=10001 euid=0 tty=/dev/pts/1 ruser=usuario user=usuario
sudo: usuario : user NOT in sudoers ; TTY=pts/1 ; PWD=/home/usuario ; COMMAND=/usr/bin/tcpdump -i any -c 5
```

`user NOT in sudoers` confirms group-based access control works: only `gerencia` members have a matching sudoers entry.

---

## 3. Windows client — pGina

The simulation result is in the [Windows client configuration](configuration.md#33-validation-by-simulation).

---

## Conclusion

Centralizing identities in OpenLDAP removed the need to create and maintain local accounts on each machine, simplified user management and kept credentials consistent. `pam_mkhomedir` reduced friction on new users' first login, and LDAP group-based sudoers enabled role-based access control with no extra local configuration.

The logs showed that every SEARCH and BIND is recorded with its result (`err`, `nentries`) and response times. That makes it possible to audit every access attempt, allowed or denied, with far better traceability than local authentication.

pGina showed that LDAP can also act as a single identity provider for Windows. It lacks the native support of Active Directory, but it works for labs and small organizations without AD DS licenses.

Improvements for a production environment:
- **LDAPS** (LDAP over TLS, port 636) to encrypt authentication traffic.
- Directory **replication** for high availability.
- **Monitoring** with real-time alerts on unauthorized access attempts.
