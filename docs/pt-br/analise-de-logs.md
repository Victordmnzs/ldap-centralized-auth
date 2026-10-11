# Análise dos logs

[← Voltar ao README](../../README.md) · [English](../en/log-analysis.md)

Os trechos completos estão em [`docs/logs/`](../logs/). Base DN, usuário de teste, IP e hostnames foram sanitizados.

---

## 1. Servidor LDAP — `journalctl -u slapd`

```bash
sudo journalctl -u slapd -n 30 --no-pager
```

Com o log em nível `stats`, dá para acompanhar a autenticação de `usuario` vinda do cliente Linux em três fases.

### 1.1 SEARCH: localizar o DN do usuário

O `nslcd` busca na árvore inteira (`scope=2`) a entrada com aquele `uid`. A primeira tentativa foi feita com o uid digitado errado:

```
conn=1007 op=6 SRCH base="dc=aluno,..." scope=2 filter="(&(objectClass=posixAccount)(uid=usuaro))"
conn=1007 op=6 SEARCH RESULT tag=101 err=0 nentries=0
```

Logo em seguida, com o uid correto:

```
conn=1007 op=7 SRCH base="dc=aluno,..." scope=2 filter="(&(objectClass=posixAccount)(uid=usuario))"
conn=1007 op=7 SEARCH RESULT tag=101 err=0 nentries=1
```

- `err=0`: a operação de busca em si não teve erro.
- `nentries=0` / `nentries=1`: nenhuma ou exatamente uma entrada encontrada.

O servidor responde com precisão também para usuários inexistentes, sem expor nada além de "nenhuma entrada".

### 1.2 BIND: validar a senha

Com o DN encontrado, o cliente abre uma nova conexão (`conn=1017`) e faz o BIND:

```
conn=1017 fd=19 ACCEPT from IP=<IP_CLIENTE>:36614
conn=1017 op=0 BIND dn="cn=Usuario Teste,ou=People,dc=aluno,dc=seg,dc=ctism,dc=ufsm,dc=br" method=128
conn=1017 op=0 BIND dn="cn=Usuario Teste,..." mech=SIMPLE bind_ssf=0 ssf=0
conn=1017 op=0 RESULT tag=97 err=0 qtime=0.000022 etime=0.000268
```

- `method=128`: mecanismo de autenticação SIMPLE (usuário e senha).
- `err=0` no `RESULT tag=97`: senha validada pelo servidor.
- `ssf=0`: sessão **sem** camada de segurança (sem TLS), o esperado num laboratório interno.
- `qtime` / `etime`: tempos de fila e de execução na casa dos microssegundos.

### 1.3 Consultas seguintes

Depois do BIND, o `nslcd` faz novos SEARCH para buscar os atributos de conta (`shadowAccount`) que o PAM precisa para concluir o login. Todos retornam `err=0`.

---

## 2. Cliente Linux — `/var/log/auth.log`

```bash
sudo tail -n 30 /var/log/auth.log
```

### 2.1 `victor` (gerencia): sudo permitido

```
sudo: victor : TTY=pts/1 ; PWD=/home/victor ; USER=root ; COMMAND=/usr/bin/tcpdump -i any -c 5
sudo: pam_unix(sudo:session): session opened for user root(uid=0)
sudo: pam_unix(sudo:session): session closed for user root
```

`victor` foi autenticado via LDAP e, como membro do grupo `gerencia` mapeado no sudoers, abriu uma sessão root para rodar o `tcpdump`. A abertura e o fechamento da sessão ficam registrados.

### 2.2 `usuario` (usuarios): home criado automaticamente

```
su: (to usuario) ldapclient on pts/1
su: pam_unix(su-l:session): session opened for user usuario(uid=10001)
```

O UID 10001, definido no diretório LDAP, é reconhecido pelo sistema local. O terminal também exibiu `Creating directory '/home/usuario'`, gerado pelo `pam_mkhomedir`.

### 2.3 `usuario` (usuarios): sudo negado

```
sudo: pam_unix(sudo:auth): authentication failure; logname=ldapclient uid=10001 euid=0 tty=/dev/pts/1 ruser=usuario user=usuario
sudo: usuario : user NOT in sudoers ; TTY=pts/1 ; PWD=/home/usuario ; COMMAND=/usr/bin/tcpdump -i any -c 5
```

`user NOT in sudoers` confirma que o controle de acesso por grupo funciona: só membros de `gerencia` têm a entrada correspondente no sudoers.

---

## 3. Cliente Windows — pGina

O resultado da simulação está na [configuração do cliente Windows](configuracao.md#33-validação-por-simulação).

---

## Conclusão

A centralização de identidades no OpenLDAP eliminou a necessidade de criar e manter contas locais em cada máquina, simplificou a gestão de usuários e garantiu credenciais consistentes. O `pam_mkhomedir` reduziu o atrito no primeiro acesso de novos usuários, e o sudoers baseado em grupo LDAP permitiu controle de acesso por perfil sem configuração local extra.

Os logs mostraram que cada SEARCH e cada BIND fica registrado com resultado (`err`, `nentries`) e tempos de resposta. Isso permite auditar cada tentativa de acesso, aprovada ou negada, com uma rastreabilidade bem maior que a da autenticação local.

O pGina mostrou que o LDAP pode servir de provedor de identidade único também para Windows. Não tem o suporte nativo de um Active Directory, mas é funcional para laboratórios e pequenas organizações sem licenças para AD DS.

Melhorias para um ambiente de produção:
- **LDAPS** (LDAP sobre TLS, porta 636) para criptografar o tráfego de autenticação.
- **Replicação** do diretório para alta disponibilidade.
- **Monitoramento** com alertas em tempo real sobre tentativas de acesso não autorizado.
