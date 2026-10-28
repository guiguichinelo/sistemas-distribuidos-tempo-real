# Diretório *LDAP* com OpenLDAP 2.6

Servidor no Docker e no WSL · cadastro, alteração e remoção de usuários, grupos e serviços · clientes Linux e Windows 11. Domínio da aula: `dc=sdtr,dc=uniube`

cliente: (troca os blocos de comandos dos experimentos 3 a 9)

Visão geral

## O que vamos montar

Um diretório com três ramos: **pessoas** (`inetOrgPerson`), **grupos** (`groupOfNames`) e **serviços** (contas de aplicação, `account + simpleSecurityObject`).

```
dc=sdtr,dc=uniube
├── ou=pessoas   → uid=ana, uid=bruno, uid=carla
├── ou=grupos    → cn=alunos, cn=monitores
└── ou=servicos  → uid=svc-wiki, uid=svc-backup
```

As duas instalações usam **Ubuntu 24.04**, cujo pacote `slapd` é o OpenLDAP 2.6.x — o resultado é idêntico no Docker e no WSL. O Docker publica a porta **1389** e o WSL usa a **389**, então os dois podem rodar juntos.

**Atenção:** LDAP puro trafega senhas em texto claro. Use só em laboratório; em produção, StartTLS/LDAPS.

Experimento 0 · professor

## Rede do WSL: deixar a sala alcançar o servidor

No WSL2 padrão (NAT) o servidor fica atrás de uma rede virtual. O modo **espelhado** (Windows 11 22H2+) faz o WSL compartilhar o IP do Windows.

```
# %UserProfile%\.wslconfig
[wsl2]
networkingMode=mirrored
```

```
# PowerShell como administrador
wsl --shutdown
New-NetFirewallRule -DisplayName "LDAP aula" -Direction Inbound -Protocol TCP -LocalPort 389,1389 -Action Allow
Set-NetFirewallHyperVVMSetting -Name '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -DefaultInboundAction Allow
ipconfig | findstr IPv4      # este é o IP que os alunos usarão
```

Sem modo espelhado, use um redirecionamento de porta:

```
$ip = (wsl hostname -I).Trim().Split(' ')[0]
netsh interface portproxy add v4tov4 listenport=389 listenaddress=0.0.0.0 connectport=389 connectaddress=$ip
```

Experimento 1 · servidor

## Docker Subindo o OpenLDAP 2.6

```
# Dockerfile
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive
RUN echo "slapd slapd/domain string sdtr.uniube" | debconf-set-selections \
 && echo "slapd shared/organization string UNIUBE" | debconf-set-selections \
 && echo "slapd slapd/password1 password admin123" | debconf-set-selections \
 && echo "slapd slapd/password2 password admin123" | debconf-set-selections \
 && echo "slapd slapd/backend select MDB" | debconf-set-selections \
 && apt-get update && apt-get install -y slapd ldap-utils \
 && rm -rf /var/lib/apt/lists/*
EXPOSE 389
CMD mkdir -p /run/slapd && chown openldap:openldap /run/slapd \
 && exec slapd -h "ldap:/// ldapi:///" -u openldap -g openldap -d 256
```

```
docker build -t ldap-aula .
docker run -d --name ldap -p 1389:389 \
  -v ldap-data:/var/lib/ldap -v ldap-config:/etc/ldap/slapd.d ldap-aula
docker exec ldap slapd -VV                      # confirma a versão 2.6.x
docker exec ldap ldapsearch -x -LLL -b dc=sdtr,dc=uniube dn
docker logs -f ldap                             # log de cada conexão e operação
```

**Observe:** o domínio `sdtr.uniube` virou o sufixo `dc=sdtr,dc=uniube` e o administrador é `cn=admin,dc=sdtr,dc=uniube`. Os volumes nomeados preservam os dados se você recriar o contêiner.

Experimento 2 · servidor

## WSL O mesmo servidor, sem contêiner

```
wsl --install -d Ubuntu-24.04      # no PowerShell, se ainda não tiver
# ---- dentro do Ubuntu ----
sudo debconf-set-selections <<'EOF'
slapd slapd/domain string sdtr.uniube
slapd shared/organization string UNIUBE
slapd slapd/password1 password admin123
slapd slapd/password2 password admin123
slapd slapd/backend select MDB
EOF
sudo DEBIAN_FRONTEND=noninteractive apt install -y slapd ldap-utils
sudo systemctl enable --now slapd      # sem systemd: sudo service slapd start
sudo ss -ltnp | grep 389
sudo slapd -VV 2>&1 | head -1
sudo journalctl -u slapd -f            # acompanhar o log
```

**Observe:** aqui o `slapd` é um serviço do systemd; no Docker ele é o processo principal em primeiro plano. Compare as duas formas de configuração: ambas guardam tudo em `/etc/ldap/slapd.d` (`cn=config`) e `/var/lib/ldap`.

Experimento 3 · cliente

## Conectar e provar que funciona

Linux Pode ser o próprio WSL, outra máquina ou uma VM. Use a porta 1389 (Docker) ou 389 (WSL).

```
sudo apt install -y ldap-utils
export H=ldap://IP_DO_SERVIDOR:1389 B=dc=sdtr,dc=uniube
export A="cn=admin,$B"  X="-x -H $H -D cn=admin,$B -w admin123"
ldapsearch -x -H $H -s base -b '' namingContexts   # anônimo: descobre o sufixo
ldapwhoami $X                                      # dn:cn=admin,dc=sdtr,dc=uniube
```

Windows 11 Sem instalar nada: o PowerShell usa a biblioteca LDAP do .NET. Salve como `ldap.ps1`:

```
using namespace System.DirectoryServices.Protocols
Add-Type -AssemblyName System.DirectoryServices.Protocols

function Connect-Ldap($Dn, $Pw, $Server = 'localhost', $Port = 1389) {
  $c = [LdapConnection]::new([LdapDirectoryIdentifier]::new($Server, $Port))
  $c.SessionOptions.ProtocolVersion = 3
  $c.AuthType = 'Basic'
  $c.Credential = [Net.NetworkCredential]::new($Dn, $Pw)
  $c.Bind(); $c
}
function Find-Ldap($c, $Base, $Filter = '(objectClass=*)', $Scope = 'Subtree', $Attrs = $null) {
  $r = [SearchRequest]::new($Base, $Filter, $Scope, [string[]]$Attrs)
  foreach ($e in $c.SendRequest($r).Entries) {
    "dn: " + $e.DistinguishedName
    foreach ($n in $e.Attributes.AttributeNames) {
      foreach ($v in $e.Attributes[$n].GetValues([string])) { "${n}: $v" } }
    ""
  }
}
function New-Ldap($c, $Dn, [hashtable]$Attrs) {
  $r = [AddRequest]::new(); $r.DistinguishedName = $Dn
  foreach ($k in $Attrs.Keys) {
    $a = [DirectoryAttribute]::new(); $a.Name = $k
    foreach ($v in @($Attrs[$k])) { [void]$a.Add([string]$v) }
    [void]$r.Attributes.Add($a) }
  [void]$c.SendRequest($r)
}
function Edit-Ldap($c, $Dn, $Op, $Attr, $Vals = @()) {   # $Op: Add | Replace | Delete
  $m = [DirectoryAttributeModification]::new(); $m.Name = $Attr; $m.Operation = $Op
  foreach ($v in $Vals) { [void]$m.Add([string]$v) }
  [void]$c.SendRequest([ModifyRequest]::new($Dn, $m))
}
function Remove-Ldap($c, $Dn) { [void]$c.SendRequest([DeleteRequest]::new($Dn)) }
```

```
. .\ldap.ps1
$B = 'dc=sdtr,dc=uniube'; $H = 'IP_DO_SERVIDOR'; $P = 1389   # 389 se for o WSL
$c = Connect-Ldap "cn=admin,$B" admin123 $H $P
Find-Ldap $c $B '(objectClass=*)' Base
```

Alternativa gráfica: o `ldp.exe`, instalado com `Add-WindowsCapability -Online -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0` (Connection → Connect → Bind simples).

**Observe:** no log do servidor aparece `conn=1000 fd=... ACCEPT`, depois `BIND` e `RESULT err=0`. Cada cliente — Linux ou Windows — fala o mesmo protocolo.

Experimento 4 · cadastrar

## Estrutura base: unidades organizacionais

```
cat > ous.ldif <<EOF
dn: ou=pessoas,$B
objectClass: organizationalUnit
ou: pessoas

dn: ou=grupos,$B
objectClass: organizationalUnit
ou: grupos

dn: ou=servicos,$B
objectClass: organizationalUnit
ou: servicos
EOF
ldapadd $X -f ous.ldif
```

```
'pessoas','grupos','servicos' | % {
  New-Ldap $c "ou=$_,$B" @{ objectClass='organizationalUnit'; ou=$_ } }
```

Experimento 5 · cadastrar

## Usuários, grupos e serviços

```
cat > pessoas.ldif <<EOF
dn: uid=ana,ou=pessoas,$B
objectClass: inetOrgPerson
uid: ana
cn: Ana Souza
givenName: Ana
sn: Souza
mail: ana@sdtr.uniube

dn: uid=bruno,ou=pessoas,$B
objectClass: inetOrgPerson
uid: bruno
cn: Bruno Lima
givenName: Bruno
sn: Lima
mail: bruno@sdtr.uniube

dn: uid=carla,ou=pessoas,$B
objectClass: inetOrgPerson
uid: carla
cn: Carla Dias
givenName: Carla
sn: Dias
mail: carla@sdtr.uniube
EOF
ldapadd $X -f pessoas.ldif
for u in ana bruno carla; do ldappasswd $X -s ${u}123 "uid=$u,ou=pessoas,$B"; done
```

```
cat > grupos.ldif <<EOF
dn: cn=alunos,ou=grupos,$B
objectClass: groupOfNames
cn: alunos
member: uid=ana,ou=pessoas,$B
member: uid=bruno,ou=pessoas,$B

dn: cn=monitores,ou=grupos,$B
objectClass: groupOfNames
cn: monitores
member: uid=carla,ou=pessoas,$B
EOF
ldapadd $X -f grupos.ldif
```

```
cat > servicos.ldif <<EOF
dn: uid=svc-wiki,ou=servicos,$B
objectClass: account
objectClass: simpleSecurityObject
uid: svc-wiki
description: Conta de leitura usada pela wiki da disciplina

dn: uid=svc-backup,ou=servicos,$B
objectClass: account
objectClass: simpleSecurityObject
uid: svc-backup
description: Conta de backup
EOF
ldapadd $X -f servicos.ldif
for s in svc-wiki svc-backup; do ldappasswd $X -s ${s}-pw "uid=$s,ou=servicos,$B"; done
```

```
$u = @{ ana=@('Ana','Souza'); bruno=@('Bruno','Lima'); carla=@('Carla','Dias') }
foreach ($k in $u.Keys) {
  New-Ldap $c "uid=$k,ou=pessoas,$B" @{
    objectClass='inetOrgPerson'; uid=$k; givenName=$u[$k][0]; sn=$u[$k][1]
    cn="$($u[$k][0]) $($u[$k][1])"; mail="$k@sdtr.uniube"; userPassword="${k}123" } }

New-Ldap $c "cn=alunos,ou=grupos,$B" @{ objectClass='groupOfNames'; cn='alunos'
  member=@("uid=ana,ou=pessoas,$B","uid=bruno,ou=pessoas,$B") }
New-Ldap $c "cn=monitores,ou=grupos,$B" @{ objectClass='groupOfNames'; cn='monitores'
  member="uid=carla,ou=pessoas,$B" }

foreach ($s in 'svc-wiki','svc-backup') {
  New-Ldap $c "uid=$s,ou=servicos,$B" @{ objectClass=@('account','simpleSecurityObject')
    uid=$s; userPassword="$s-pw"; description="Conta de serviço $s" } }
```

**Testar a autenticação** de um usuário e de um serviço:

```
ldapwhoami -x -H $H -D uid=ana,ou=pessoas,$B -w ana123
ldapwhoami -x -H $H -D uid=svc-wiki,ou=servicos,$B -w svc-wiki-pw
ldapwhoami -x -H $H -D uid=ana,ou=pessoas,$B -w errada   # Invalid credentials (49)
```

```
Connect-Ldap "uid=ana,ou=pessoas,$B" ana123 $H $P | Out-Null; 'ana autenticou'
Connect-Ldap "uid=svc-wiki,ou=servicos,$B" svc-wiki-pw $H $P | Out-Null; 'serviço autenticou'
Connect-Ldap "uid=ana,ou=pessoas,$B" errada $H $P     # LdapException: credenciais inválidas
```

**Observe:** no Linux, `ldappasswd` faz o *servidor* guardar a senha como hash `{SSHA}`. No PowerShell a senha foi enviada como atributo comum e fica em texto claro — confira no experimento 6 e discuta por quê.

Experimento 6 · consultar

## Buscas: base, escopo e filtro

```
ldapsearch $X -LLL -b $B dn                                  # árvore inteira
ldapsearch $X -LLL -b ou=pessoas,$B -s one cn mail          # só um nível
ldapsearch $X -LLL -b $B '(&(objectClass=inetOrgPerson)(mail=*@sdtr.uniube))' uid
ldapsearch $X -LLL -b $B '(uid=a*)' cn
ldapsearch $X -LLL -b ou=grupos,$B '(member=uid=ana,ou=pessoas,'$B')' cn   # grupos da Ana
ldapsearch $X -LLL -b $B '(uid=ana)' userPassword           # hash {SSHA}
```

```
Find-Ldap $c $B '(objectClass=*)' Subtree 'dn'
Find-Ldap $c "ou=pessoas,$B" '(objectClass=*)' OneLevel 'cn','mail'
Find-Ldap $c $B '(&(objectClass=inetOrgPerson)(mail=*@sdtr.uniube))' Subtree 'uid'
Find-Ldap $c $B '(uid=a*)' Subtree 'cn'
Find-Ldap $c "ou=grupos,$B" "(member=uid=ana,ou=pessoas,$B)" Subtree 'cn'   # grupos da Ana
Find-Ldap $c $B '(uid=ana)' Subtree 'userPassword'          # texto claro!
```

**Observe:** **base** (onde começa), **escopo** (base / one / sub) e **filtro** definem a busca. Os grupos guardam os *membros*; descobrir “a que grupos a Ana pertence” é uma busca reversa por `member`.

Experimento 7 · alterar

## Modificar atributos e participação em grupos

```
ldapmodify $X <<EOF
dn: uid=ana,ou=pessoas,$B
changetype: modify
replace: mail
mail: ana.souza@sdtr.uniube
-
add: telephoneNumber
telephoneNumber: +55 34 99999-0001
-
delete: givenName

dn: cn=monitores,ou=grupos,$B
changetype: modify
add: member
member: uid=bruno,ou=pessoas,$B
EOF
ldappasswd $X -s novaSenha1 "uid=carla,ou=pessoas,$B"      # redefinir senha
ldapsearch $X -LLL -b $B '(|(uid=ana)(cn=monitores))'
```

```
$ana = "uid=ana,ou=pessoas,$B"
Edit-Ldap $c $ana Replace mail 'ana.souza@sdtr.uniube'
Edit-Ldap $c $ana Add telephoneNumber '+55 34 99999-0001'
Edit-Ldap $c $ana Delete givenName
Edit-Ldap $c "cn=monitores,ou=grupos,$B" Add member "uid=bruno,ou=pessoas,$B"
Edit-Ldap $c "uid=carla,ou=pessoas,$B" Replace userPassword 'novaSenha1'
Find-Ldap $c $B '(|(uid=ana)(cn=monitores))'
```

**Observe:** as três operações do `modify` — **add**, **replace**, **delete** — e que tentar *add* de um valor que já existe falha (`Type or value exists`, err=20). Rode duas vezes para ver.

Experimento 8 · remover

## Apagar com cuidado: a ordem importa

```
# 1) tirar a Carla do grupo ANTES de apagar a conta
ldapmodify $X <<EOF
dn: cn=monitores,ou=grupos,$B
changetype: modify
delete: member
member: uid=carla,ou=pessoas,$B
EOF
ldapdelete $X "uid=carla,ou=pessoas,$B"          # 2) apagar o usuário
ldapdelete $X "uid=svc-backup,ou=servicos,$B"    # 3) apagar um serviço
ldapdelete $X "ou=servicos,$B"                   # erro: not allowed on non-leaf (66)
ldapdelete $X -r "ou=servicos,$B"                # -r apaga a subárvore inteira
ldapdelete $X "cn=monitores,ou=grupos,$B"        # 4) apagar um grupo
```

```
Edit-Ldap $c "cn=monitores,ou=grupos,$B" Delete member "uid=carla,ou=pessoas,$B"
Remove-Ldap $c "uid=carla,ou=pessoas,$B"
Remove-Ldap $c "uid=svc-backup,ou=servicos,$B"
Remove-Ldap $c "ou=servicos,$B"                  # erro: NotAllowedOnNonLeaf
Remove-Ldap $c "uid=svc-wiki,ou=servicos,$B"     # apague os filhos primeiro...
Remove-Ldap $c "ou=servicos,$B"                  # ...e então a unidade
Remove-Ldap $c "cn=monitores,ou=grupos,$B"
```

**Atenção:** sem o overlay `refint`, apagar um usuário *não* remove sua entrada `member` dos grupos — o grupo fica com uma referência órfã. Teste: apague o Bruno sem tirá-lo do grupo `alunos` e consulte o grupo.

Experimento 9 · permissões

## O que um usuário comum pode fazer

A configuração padrão do pacote deixa cada pessoa trocar a própria senha, impede ler hashes de senha e proíbe alterar entradas alheias.

```
AN="-x -H $H -D uid=ana,ou=pessoas,$B -w ana123"
ldapsearch $AN -LLL -b $B '(uid=bruno)' mail userPassword   # lê o mail, não vê a senha
ldapmodify $AN <<EOF
dn: uid=bruno,ou=pessoas,$B
changetype: modify
replace: mail
mail: invasor@x.com
EOF
# → Insufficient access (50)
ldappasswd $AN -s nova456                                   # a Ana troca a PRÓPRIA senha
```

```
$a = Connect-Ldap "uid=ana,ou=pessoas,$B" ana123 $H $P
Find-Ldap $a $B '(uid=bruno)' Subtree 'mail','userPassword'   # sem userPassword
Edit-Ldap $a "uid=bruno,ou=pessoas,$B" Replace mail 'invasor@x.com'   # InsufficientAccessRights
Edit-Ldap $a "uid=ana,ou=pessoas,$B" Replace userPassword 'nova456'   # permitido
```

**Observe:** o administrador (`rootDN`) ignora as ACLs; todos os outros passam por elas. Para ver as regras: `ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config olcAccess` (rodando como root no servidor).

Fechamento

## Desafios e backup

- **Desafio 1:** cada aluno cadastra a própria conta (uid = seu nome) e entra no grupo `alunos` — pelo cliente do próprio notebook, com o servidor do professor.
- **Desafio 2:** o `svc-wiki` faz bind e lista só os e-mails de `ou=pessoas`.
- **Desafio 3:** parar o servidor, recriá-lo e provar (com os volumes ou o `/var/lib/ldap`) que os dados continuam lá.

```
# Backup em LDIF
docker exec ldap slapcat -b dc=sdtr,dc=uniube > backup.ldif     # Docker
sudo slapcat -b dc=sdtr,dc=uniube > backup.ldif                  # WSL
```

| Operação | Linux | Windows (ldap.ps1) |
| --- | --- | --- |
| buscar | ldapsearch | Find-Ldap |
| cadastrar | ldapadd | New-Ldap |
| alterar | ldapmodify / ldappasswd | Edit-Ldap |
| remover | ldapdelete | Remove-Ldap |

Aula de OpenLDAP 2.6 · Docker + WSL · clientes Linux e Windows 11
