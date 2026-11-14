# Guia do Trabalho Final — Redes de Computadores 2

Domínio: `g07rc2.com.br` · Debian (servidores em modo terminal, cliente com interface gráfica)

Este guia segue o **passo a passo dos slides do professor** (aulas 2 a 6). Onde o trabalho exige algo diferente dos slides, a seção avisa com a marcação **[ADAPTADO]**. O que não existe nos slides (HTTPS, CA, e-mail, SSH por chave) vem marcado como **[EXTRA]**.

> Convenções dos slides: todos os comandos de servidor rodam como **root** (por isso o ajuste do PATH da seção 1.1). Reinicie serviços com `service <nome> restart`. Cada etapa termina com uma **Evidência**: é o que o professor quer ver (e o que você deve copiar para o relatório).

> **Marcações de máquina.** Cada seção diz em qual máquina o comando roda: **[INFRA]**, **[E-MAIL]** ou **[CLIENTE]**. Os blocos **➜ Reflexo em outras máquinas** avisam o que precisa ser ajustado em seguida, em outra máquina, quando você mexe naquele ponto. A tabela "Se você mexer em..." da seção 0 reúne tudo isso num só lugar.

---

## 0. O que muda em relação aos slides

| Slides | Trabalho | O que fazer |
|---|---|---|
| Rede `192.168.10.0/24` (Classe C) | O trabalho exige rede interna **Classe B** | Usar `172.16.0.0/16` (máscara `255.255.0.0`) no lugar |
| Domínio `disciplinacsr.com.br` | Domínio do grupo: `g07rc2.com.br` | Trocar em DNS, Apache e e-mail |
| Zona reversa `10.168.192.in-addr.arpa` (/24) | Rede /16 | Zona reversa `16.172.in-addr.arpa` e PTR com **dois octetos** (`1.0`, `2.0`) |
| DHCP entrega IP, gateway e DNS | O trabalho também pede **domínio de busca** | Acrescentar `option domain-name` e `option domain-search` |
| SSH com senha | Trabalho pede **autenticação por chave** | `ssh-keygen` + `ssh-copy-id` e desativar senha **[EXTRA]** |
| Não há | Apache, HTTPS, CA local, e-mail | Seções 1.8, 1.9 e 2 **[EXTRA]** |

### Plano de endereçamento

| Máquina | Hostname (nome da máquina) | Nome no DNS | Interface | IP |
|---|---|---|---|---|
| Servidor de infraestrutura | `infra` | `infra.g07rc2.com.br` (e também `www` e `intranet`) | `enp0s3` (externa/NAT da VM) | DHCP da VM |
| | | | `enp0s8` (interna) | `172.16.0.1/16` (fixo) |
| Servidor de e-mail | `mail` | `mail.g07rc2.com.br` | interna | `172.16.0.2/16` (fixo) |
| Estação cliente | nome padrão (não muda) | não precisa de registro | interna | DHCP, faixa `172.16.1.100–172.16.1.200` |

- Gateway da rede interna: `172.16.0.1` · DNS: `172.16.0.1` · Broadcast: `172.16.255.255`
- Reserva por MAC: o cliente fica com `172.16.1.50`.

### VMs (VirtualBox ou similar)

- **Infra**: 2 placas. Adaptador 1 = NAT. Adaptador 2 = Rede interna (ex.: `redeint`).
- **E-mail** e **Cliente**: 1 placa cada, na mesma Rede interna `redeint`.

### Nomes das máquinas (hostname) e nomes do DNS

São duas coisas separadas:

- **Hostname** é o nome que a máquina usa para si mesma (comando `hostname`).
- **Nome no DNS** é um registro da zona (`www`, `intranet`, `mail`, o MX...) que aponta para um IP.

O trabalho cobra os nomes do **DNS**. Mesmo assim, vamos **renomear as duas máquinas servidoras** para `infra` e `mail` (o cliente fica com o nome que já tem), porque assim o hostname bate com o DNS, com o `myhostname` do Postfix e com os certificados. Em cada máquina o nome é definido em **dois lugares**: `hostnamectl set-hostname <nome-curto>` e a linha `127.0.1.1   <nome-curto>.g07rc2.com.br <nome-curto>` em `/etc/hosts`. Confira com `hostname` (nome curto) e `hostname -f` (nome completo).

| Máquina | Comando | Linha no `/etc/hosts` | Onde esse nome aparece depois |
|---|---|---|---|
| **[INFRA]** | `hostnamectl set-hostname infra` | `127.0.1.1   infra.g07rc2.com.br infra` | SOA, NS e registro A na zona DNS (1.6) |
| **[E-MAIL]** | `hostnamectl set-hostname mail` | `127.0.1.1   mail.g07rc2.com.br mail` | `myhostname` do Postfix (2.4), registro A e MX no DNS (1.6), certificado (1.8) |

> Se preferir manter `servidor1`/`servidor2`, também funciona: o trabalho não exige renomear. Nesse caso, troque o nome só nos lugares da última coluna.

### Ordem recomendada (com o que cada etapa precisa antes)

| # | Máquina | Seções | Precisa estar pronto antes | Depois disso, outra máquina pode... |
|---|---|---|---|---|
| 1 | **[INFRA]** | 1.1 e 1.2: nome, rede e reboot | VM com 2 placas | Usar `172.16.0.1` como gateway e DNS |
| 2 | **[INFRA]** | 1.4: NAT | 1.2 | **[E-MAIL]** e **[CLIENTE]** passam a fazer `apt-get` e `ping 8.8.8.8` |
| 3 | **[INFRA]** | 1.3 DHCP e 1.6 DNS | 1.2 | **[CLIENTE]** pega IP, gateway e DNS |
| 4 | **[CLIENTE]** | 3.1 e 3.2: rede por DHCP e ferramentas | 1.3 e 1.4 | Testar DHCP e `dig`, e acessar a infra por SSH e Samba |
| 5 | **[INFRA]** | 1.5 SSH e 1.7 Samba | Cliente com IP (etapa 4) | Testar SSH e Samba a partir do cliente |
| 6 | **[INFRA]** | 1.8 CA e 1.9 Apache | 1.6 (nomes `www` e `intranet` no DNS) | Gerar `mail.crt`/`mail.key` e `ca.crt` para distribuir |
| 7 | **[E-MAIL]** | 2.1 e 2.2: nome, rede, pacotes e `ssh` | 1.4 (internet) e 1.6 (DNS) | Receber o certificado por `scp` |
| 8 | **[INFRA]** e **[E-MAIL]** | 2.3 a 2.7: certificado, Postfix, Dovecot | 1.8 e etapa 7 | **[CLIENTE]** configura a conta de e-mail |
| 9 | **[CLIENTE]** | 3.3 a 3.5: CA, testes, Thunderbird | Etapas 6 e 8 | Demonstração completa |
| 10 | Todas | Capturas, repositório, relatório | Tudo funcionando após reboot | |

### Se você mexer em..., confira também

Esta é a tabela para quando algo muda no meio do caminho. Ela diz o que mexer em seguida e em qual máquina.

| Se você mexer em... | Na máquina | Então confira também... |
|---|---|---|
| IP da infra (`172.16.0.1`) | **[INFRA]** | DHCP: `routers` e `domain-name-servers` (1.3). DNS: registros A de `infra`, `www`, `intranet` e PTR `1.0` (1.6). **[E-MAIL]**: `gateway` e `nameserver` (2.1). **[CLIENTE]**: comandos de SSH e Samba |
| IP do e-mail (`172.16.0.2`) | **[E-MAIL]** | DNS: registro A de `mail` e PTR `2.0` (1.6). **[CLIENTE]**: `ssh-copy-id` e `ssh` para esse IP |
| Hostname de uma máquina | A própria | `/etc/hosts` dela. Se for o e-mail: `myhostname` do Postfix e DNS. Se for a infra: SOA e NS da zona |
| Nome de serviço (`www`, `intranet`, `mail`) | **[INFRA]** | Zona DNS (aumentar o serial). `ServerName` do Apache (1.9). `subjectAltName` do certificado, que precisa ser **refeito e reinstalado** (1.8). **[CLIENTE]**: conta do Thunderbird |
| Qualquer arquivo de zona do DNS | **[INFRA]** | Serial maior, `named-checkzone`, `service bind9 restart`. **[CLIENTE]**: testar com `dig` |
| Rede ou faixa do DHCP | **[INFRA]** | `-s` da regra NAT (1.4), zona reversa (1.6). **[E-MAIL]**: IP e máscara (2.1) |
| Nome da interface (`enp0s3`, `enp0s8`) | Cada máquina | `interfaces`, `INTERFACESv4` (1.3), `-o` do NAT (1.4), `tcpdump -i` (seção 4) |
| CA ou certificados | **[INFRA]** | Reinstalar `web.crt`/`web.key` para o Apache. Copiar `mail.crt`/`mail.key` para o **[E-MAIL]**. Reimportar a CA no **[CLIENTE]**. Reiniciar Apache, Postfix e Dovecot |
| Contas de e-mail (`ana`, `bia`) | **[E-MAIL]** | **[CLIENTE]**: contas do Thunderbird |
| Chave SSH do cliente | **[CLIENTE]** | `ssh-copy-id` para **cada** servidor, testar, e só depois `PasswordAuthentication no` em cada um |

---

## 1. Servidor de infraestrutura

### 1.1 Preparação do ambiente (Aula 2) **[INFRA]**

Instale as ferramentas de diagnóstico:

```bash
apt-get install net-tools iptables iptables-persistent vim rsyslog
```

Ajuste o PATH do root. Abra o arquivo e adicione a **linha final**:

```bash
vim /root/.bashrc
```

```
export PATH=$PATH:/sbin:/bin:/usr/sbin
```

**Nome da máquina.** Esta máquina passa a se chamar `infra`:

```bash
hostnamectl set-hostname infra
```

Em `/etc/hosts`, mantenha a linha `127.0.0.1 localhost` e substitua a linha `127.0.1.1` que já existe por:

```
127.0.1.1   infra.g07rc2.com.br infra
```

O prompt só mostra `infra` em um terminal novo (o `reboot` de 1.2 já resolve).

**Evidência:** `hostname` (mostra `infra`), `hostname -f` (mostra `infra.g07rc2.com.br`), `whoami`, `pwd`.

> **➜ Reflexo em outras máquinas:** nenhum agora. Esse nome só precisa bater com o que a zona DNS usa para a infra (`infra` no SOA, no NS e no registro A, em 1.6), e a zona deste guia já está assim.

### 1.2 Configuração de rede (Aula 2) **[ADAPTADO: Classe B]** **[INFRA]**

Antes de editar, confirme os nomes das interfaces (o nome pode mudar conforme a VM):

```bash
ip a
```

Edite `/etc/network/interfaces`:

```
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet dhcp

allow-hotplug enp0s8
iface enp0s8 inet static
address 172.16.0.1
netmask 255.255.0.0
```

Reinicie:

```bash
reboot
```

**Evidência:** `ip a` deve mostrar `enp0s3` com IP dinâmico (`scope global dynamic`) e `enp0s8` com `172.16.0.1/16`.

> **➜ Reflexo em outras máquinas:** os nomes das interfaces voltam em três lugares da **infra**: `INTERFACESv4` (1.3), `-o enp0s3` do NAT (1.4) e `tcpdump -i` (seção 4). Se a sua VM usa outros nomes, troque nos três. Já o IP `172.16.0.1` é o gateway e o DNS de todo mundo: aparece no DHCP (1.3), no DNS (1.6), no **[E-MAIL]** (2.1) e nos comandos do **[CLIENTE]**.

### 1.3 DHCP (Aula 3) **[ADAPTADO: Classe B + domínio de busca]** **[INFRA]**

Instale:

```bash
apt-get install isc-dhcp-server
```

Defina a interface de escuta. Edite `/etc/default/isc-dhcp-server`:

```
INTERFACESv4="enp0s8"
INTERFACESv6=""
```

Faça o backup do arquivo original e crie um novo:

```bash
mv /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.original
vim /etc/dhcp/dhcpd.conf
```

Conteúdo do `/etc/dhcp/dhcpd.conf`:

```
option domain-name "g07rc2.com.br";
option domain-search "g07rc2.com.br";
option domain-name-servers 172.16.0.1, 8.8.8.8;

default-lease-time 600;
max-lease-time 7200;
ddns-update-style none;

subnet 172.16.0.0 netmask 255.255.0.0 {
  range 172.16.1.100 172.16.1.200;

  option domain-name-servers 172.16.0.1, 8.8.8.8;
  option routers 172.16.0.1;
  option broadcast-address 172.16.255.255;
}

# Reserva por MAC (troque pelo MAC real do cliente)
host cliente {
  hardware ethernet 08:00:27:AA:14:80;
  fixed-address 172.16.1.50;
}
```

> A reserva (`host`) deve ficar no mesmo arquivo, fora do bloco `subnet`, como no slide 15 da aula.
>
> Para ver o MAC real do cliente: `ip a` na máquina cliente (linha `link/ether`).

Reinicie e habilite:

```bash
service isc-dhcp-server restart
systemctl enable isc-dhcp-server
```

> Logo depois da instalação o serviço pode falhar ao subir. Isso é normal enquanto o `dhcpd.conf` ainda não foi configurado.

**Evidência (no cliente):** `ip a` e `ping 172.16.0.1`. O cliente deve receber IP dentro da faixa (ou o `172.16.1.50` da reserva).
**Evidência (no servidor):** `ip a show enp0s8` e `ping 172.16.1.50`.

> **➜ Reflexo em outras máquinas:**
> - **[CLIENTE]**: a placa precisa estar na mesma Rede interna e em DHCP automático (3.1). O MAC da reserva sai de `ip a` no cliente (linha `link/ether`).
> - O que o DHCP entrega precisa bater com o resto: `routers` = IP da infra, `domain-name-servers` = a máquina com o BIND (1.6), `domain-search` = domínio da zona.
> - `host cliente` é só um rótulo da reserva. Vale o MAC, não o hostname do cliente.
> - Se mudar a rede ou a faixa, ajuste também o `-s` do NAT (1.4) e a zona reversa (1.6).

### 1.4 Roteamento, NAT e persistência (Aula 3) **[INFRA]**

Habilite o encaminhamento de pacotes. Edite `/etc/sysctl.conf` e deixe ativa a linha (no slide, é a linha 28):

```
net.ipv4.ip_forward=1
```

Aplique:

```bash
sysctl -p
```

Regra de NAT (interface de saída = a **externa**, `enp0s3`) **[ADAPTADO: rede]**:

```bash
iptables -t nat -A POSTROUTING -o enp0s3 -s 172.16.0.0/16 -j MASQUERADE
```

Salve a regra para sobreviver ao reboot e reinicie:

```bash
iptables-save > /etc/iptables/rules.v4
reboot
```

**Evidência:** depois do reboot,

```bash
cat /etc/iptables/rules.v4
```

deve conter:

```
-A POSTROUTING -s 172.16.0.0/16 -o enp0s3 -j MASQUERADE
```

Do cliente, `ping 8.8.8.8` deve responder (se falhar: `ip_forward`, regra NAT e interface `enp0s3`).

> **➜ Reflexo em outras máquinas:** só depois deste passo o **[E-MAIL]** e o **[CLIENTE]** conseguem rodar `apt-get` e `ping 8.8.8.8`. Se mudar a rede interna, troque o `-s` da regra, rode `iptables-save > /etc/iptables/rules.v4` de novo e reinicie para conferir.

### 1.5 SSH (Aula 4) **[INFRA]** e **[CLIENTE]**

Instale e valide (no **servidor**):

```bash
apt-get install ssh
systemctl status ssh
netstat -tanp | grep :22
```

Política de acesso em `/etc/ssh/sshd_config` (os slides mostram estas diretivas):

```
Port 22
LoginGraceTime 120
PermitRootLogin no
PermitEmptyPasswords no
```

Reinicie:

```bash
service ssh restart
```

Primeira conexão, **a partir do cliente** (o comando roda no cliente, mas abre um terminal no servidor):

```bash
ssh aluno@172.16.0.1
```

(Aceite a chave do host uma vez; ela fica em `~/.ssh/known_hosts`.)

Prove que a sessão é remota: dentro da sessão, `hostname`, `whoami`, `pwd`; depois:

```bash
echo "Aula 04 SSH" > ~/aula04-ssh.txt
exit
ssh aluno@172.16.0.1
cat ~/aula04-ssh.txt
```

**Autenticação por chave [EXTRA]**: o trabalho pede chave. No **cliente**:

```bash
ssh-keygen -t ed25519
ssh-copy-id aluno@172.16.0.1
ssh-copy-id aluno@172.16.0.2     # só depois de 2.2 (lá o usuário aluno é criado)
ssh aluno@172.16.0.1              # tem que entrar sem pedir senha
```

Só depois de testar, no servidor, edite `/etc/ssh/sshd_config`:

```
PubkeyAuthentication yes
PasswordAuthentication no
```

```bash
sshd -t
service ssh restart
```

> Mantenha uma sessão SSH aberta enquanto reinicia, para não se trancar para fora.
> Se quiser porta alternativa (como na avaliação prática), troque `Port` e use `ssh -p 8822 aluno@172.16.0.1`. A porta do servidor e a do cliente precisam combinar.

**Erros comuns (slide 18):** `Connection refused` = serviço parado/porta fechada; `Connection timed out` = IP/rede errada ou VM desligada; `Permission denied` = usuário, senha ou chave errados.

> **➜ Reflexo em outras máquinas:** a chave é gerada **no cliente** e precisa ser copiada para **cada** servidor (`ssh-copy-id` para a infra agora e para o e-mail depois de 2.2). Teste a entrada sem senha em cada servidor antes de desativar a senha nele, e faça isso um servidor por vez. A `Port` do `sshd_config` e o `ssh -p` do cliente precisam combinar.

### 1.6 DNS com BIND9 (Aula 6) **[ADAPTADO: domínio e Classe B]** **[INFRA]**

Instale:

```bash
apt-get update
apt-get install bind9 dnsutils
```

**Encaminhadores (opcional).** Faça backup e edite as opções:

```bash
cp /etc/bind/named.conf.options /etc/bind/named.conf.options.bkp
vim /etc/bind/named.conf.options
```

Dentro de `options { ... }`:

```
forwarders {
    8.8.8.8;
};
```

**Declare as zonas** em `/etc/bind/named.conf.local`:

```
zone "g07rc2.com.br" IN {
  type primary;
  file "/etc/bind/db.g07rc2.com.br";
};

zone "16.172.in-addr.arpa" {
  type primary;
  file "/etc/bind/db.172.16.rev";
};
```

**Arquivo da zona direta.** Copie o ponto de partida (`db.local`) e edite:

```bash
cp /etc/bind/db.local /etc/bind/db.g07rc2.com.br
vim /etc/bind/db.g07rc2.com.br
```

Conteúdo final de `/etc/bind/db.g07rc2.com.br`:

```
$TTL    604800
@   IN  SOA infra.g07rc2.com.br. root.g07rc2.com.br. (
            2026111001  ; Serial
            604800      ; Refresh
            86400       ; Retry
            2419200     ; Expire
            604800 )    ; Negative Cache TTL
;
@         IN  NS   infra.g07rc2.com.br.
@         IN  MX   10 mail.g07rc2.com.br.
@         IN  A    172.16.0.1
infra     IN  A    172.16.0.1
mail      IN  A    172.16.0.2
www       IN  A    172.16.0.1
intranet  IN  A    172.16.0.1
```

> O alvo do MX deve ser um registro **A** (nunca um CNAME).
> **Aumente o serial** sempre que alterar o conteúdo da zona.

**Arquivo da zona reversa.** Copie `db.127` e edite:

```bash
cd /etc/bind
cp db.127 db.172.16.rev
vim db.172.16.rev
```

Conteúdo final de `/etc/bind/db.172.16.rev`:

```
$TTL    604800
@   IN  SOA infra.g07rc2.com.br. root.g07rc2.com.br. (
            2026111001  ; Serial
            604800      ; Refresh
            86400       ; Retry
            2419200     ; Expire
            604800 )    ; Negative Cache TTL
;
@      IN  NS   infra.g07rc2.com.br.
1.0    IN  PTR  infra.g07rc2.com.br.
2.0    IN  PTR  mail.g07rc2.com.br.
```

> No slide (rede /24) o PTR usa só o último octeto (`1`). Como aqui a rede é /16, o PTR usa **os dois últimos octetos invertidos**: o IP `172.16.0.2` vira `2.0`.

**Valide antes de reiniciar:**

```bash
named-checkconf
named-checkzone g07rc2.com.br /etc/bind/db.g07rc2.com.br
named-checkzone 16.172.in-addr.arpa /etc/bind/db.172.16.rev
```

**Reinicie:**

```bash
service bind9 restart
systemctl enable bind9
```

Se não iniciar: `journalctl -u bind9 -n 40`.

O próprio servidor deve usar o seu DNS. Em `/etc/resolv.conf`:

```
search g07rc2.com.br
nameserver 127.0.0.1
```

**Testes (do cliente):**

```bash
dig www.g07rc2.com.br
dig intranet.g07rc2.com.br
dig g07rc2.com.br MX
dig -x 172.16.0.2
dig naoexiste.g07rc2.com.br        # status: NXDOMAIN
```

**Evidência esperada:** `A = 172.16.0.1`, `MX = mail.g07rc2.com.br.`, `PTR = mail.g07rc2.com.br.`.
Para explicar o `NXDOMAIN` na defesa: o servidor é autoritativo da zona e responde que o nome **não existe** (o SOA vem junto e define o tempo de cache negativo).

**Diagnóstico (slide 21):** sem resposta = `bind9`, rede e porta 53 UDP/TCP; `SERVFAIL` = sintaxe, serial, logs; `PTR errado` = zona reversa e octetos do IP. Se o cliente receber `REFUSED`, acrescente em `options { ... }`: `allow-recursion { 172.16.0.0/16; };`.

> **➜ Reflexo em outras máquinas:**
> - Mexeu na zona? Aumente o serial, rode `named-checkzone` e reinicie o `bind9` (na **infra**).
> - Nome novo ou alterado: precisa existir também no `ServerName` do Apache (1.9) e no `subjectAltName` do certificado (1.8, que precisa ser refeito e reinstalado).
> - IP do `mail` alterado: ajuste o IP fixo do **[E-MAIL]** (2.1) e o PTR `2.0`.
> - **[CLIENTE]**: teste com `dig`. Se o nome do e-mail mudar, refaça a conta do Thunderbird (3.5).
> - Depois de reiniciar a infra, confira `cat /etc/resolv.conf`: se o `dhclient` da `enp0s3` sobrescrever o arquivo, reponha `search` e `nameserver 127.0.0.1`.

### 1.7 Samba (Aula 5) **[INFRA]** e **[CLIENTE]**

> O trabalho cita "permissões do compartilhamento" na validação (passo 3). Faça pelo menos a versão com grupo e usuário específico.

Instale e faça backup da configuração:

```bash
apt-get update
apt-get install samba smbclient cifs-utils
cp /etc/samba/smb.conf /etc/samba/smb.conf.bkp
```

**Usuários.** O usuário precisa existir no Linux e depois ser cadastrado no Samba (são senhas diferentes):

```bash
adduser aluno
smbpasswd -a aluno
adduser professor
smbpasswd -a professor
```

**Pastas e grupo:**

```bash
mkdir -p /mnt/Publico /mnt/grupos /mnt/usuario

groupadd ifsulalunos
gpasswd -a aluno ifsulalunos

chmod -R 777 /mnt/Publico            # atalho do laboratório (aberto de propósito)
chgrp ifsulalunos /mnt/grupos
chmod 770 /mnt/grupos
chown professor /mnt/usuario
chmod 700 /mnt/usuario
```

> O slide mostra `chmod -R 777` como atalho de laboratório e discute que a permissão correta segue quem precisa ler, escrever e executar. Os `chgrp/chown/chmod` acima são para o Linux também respeitar a regra do Samba.

**Edite `/etc/samba/smb.conf`** (seções do slide):

```
[global]
   workgroup = RC2
   netbios name = Samba-Server
   server string = %h servidor de arquivos
   security = user
   log file = /var/log/samba/%m.%u.log
   max log size = 100000

[homes]
   valid users = %S
   read only = no
   browseable = yes
   create mask = 0700
   directory mask = 0700

[Dados_Publicos]
   comment = Diretorio Publico Livre
   available = yes
   browseable = yes
   path = /mnt/Publico
   public = yes
   writable = yes

[diretorio_grupos]
   path = /mnt/grupos
   writable = yes
   valid users = +ifsulalunos

[diretorio_usuario]
   path = /mnt/usuario
   writable = yes
   valid users = professor
```

> O sinal `+` indica **grupo** autorizado. `public = yes` com `writable = yes` serve para demonstrar, mas exige cuidado fora do laboratório.

**Valide e reinicie:**

```bash
testparm
service smbd restart
service nmbd restart
systemctl enable smbd nmbd
systemctl status smbd
ss -tlnp | grep -E '139|445'
```

**Teste a partir do cliente** (o teste real é fora do servidor):

```bash
smbclient //172.16.0.1/Dados_Publicos -U aluno
smbclient //172.16.0.1/diretorio_grupos -U aluno         # permitido (está no grupo)
smbclient //172.16.0.1/diretorio_grupos -U professor     # negado (fora do grupo)
smbclient //172.16.0.1/diretorio_usuario -U professor    # permitido
smbclient //172.16.0.1/diretorio_usuario -U aluno        # negado
```

Pelo gerenciador de arquivos do cliente: acessar `smb://172.16.0.1`, depois informar usuário, domínio (`RC2`) e senha.

**Montagem automática no cliente (slide 20):**

```bash
apt-get install cifs-utils vim
vim /home/aluno/.samba
```

```
user=aluno
password=aluno
domain=RC2
```

```bash
chmod 600 /home/aluno/.samba
mkdir -p /mnt/aluno
```

Em `/etc/fstab` (cliente):

```
//172.16.0.1/homes   /mnt/aluno   cifs   defaults,credentials=/home/aluno/.samba   0   0
```

```bash
mount -a
```

> O arquivo `.samba` tem senha em texto claro. O `chmod 600` não é opcional, e esse arquivo **não vai para o repositório**.
> Se a VM demorar no boot esperando a rede, use `defaults,_netdev,nofail,credentials=...`.

**Logs (evidência):** `/var/log/samba/`.

> **➜ Reflexo em outras máquinas:** os usuários `aluno` e `professor` e o grupo `ifsulalunos` existem só na **infra**. O **[CLIENTE]** apenas consome (`smbclient`, `fstab`) pelo IP `172.16.0.1`. Usuário novo no Samba precisa de `smbpasswd -a` (a senha do Samba é separada da do Linux).

### 1.8 Autoridade certificadora (CA) local **[EXTRA]** **[INFRA]**

No servidor de infra, em `~/ca`:

```bash
apt-get install openssl ssl-cert
mkdir -p ~/ca && cd ~/ca

openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/C=BR/O=g07rc2/CN=CA Laboratorio g07rc2" -out ca.crt
```

**Certificado dos sites (www + intranet):**

```bash
openssl genrsa -out web.key 2048
openssl req -new -key web.key -subj "/CN=www.g07rc2.com.br" -out web.csr

cat > web.ext <<'EOF'
subjectAltName = DNS:www.g07rc2.com.br, DNS:intranet.g07rc2.com.br
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
EOF

openssl x509 -req -in web.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out web.crt -days 825 -sha256 -extfile web.ext
```

**Certificado do e-mail:**

```bash
openssl genrsa -out mail.key 2048
openssl req -new -key mail.key -subj "/CN=mail.g07rc2.com.br" -out mail.csr

cat > mail.ext <<'EOF'
subjectAltName = DNS:mail.g07rc2.com.br
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
EOF

openssl x509 -req -in mail.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out mail.crt -days 825 -sha256 -extfile mail.ext
```

Instale o certificado dos sites no servidor de infra:

```bash
cp web.crt /etc/ssl/certs/web.crt
cp web.key /etc/ssl/private/web.key
chmod 640 /etc/ssl/private/web.key
chgrp ssl-cert /etc/ssl/private/web.key
```

Deixe a CA **pública** (só o `.crt`) disponível para o cliente buscar depois (3.3):

```bash
cp ~/ca/ca.crt /home/aluno/ca.crt
chown aluno:aluno /home/aluno/ca.crt
```

> **Nunca** suba `*.key` (privadas) para o repositório. Use `.gitignore` para `*.key` e `.htpasswd`.

> **➜ Reflexo em outras máquinas:** este passo gera três coisas que vão para lugares diferentes:
> - `web.crt` e `web.key`: ficam na **infra** e são usados pelo Apache (1.9).
> - `mail.crt` e `mail.key`: precisam ser copiados para o **[E-MAIL]** (2.3).
> - `ca.crt`: precisa ser instalado no **[CLIENTE]** e importado no Firefox e no Thunderbird (3.3).
>
> Se você refizer a CA ou qualquer certificado, repita as três cópias e reinicie Apache, Postfix e Dovecot. O `subjectAltName` precisa listar todos os nomes do site: se um nome mudar no DNS, refaça o certificado.

### 1.9 Apache com dois sites (hosts virtuais) **[EXTRA]** **[INFRA]**

```bash
apt-get install apache2 apache2-utils
a2enmod ssl rewrite headers
mkdir -p /var/www/www /var/www/intranet
echo "<h1>Site publico g07rc2</h1>" > /var/www/www/index.html
echo "<h1>Intranet g07rc2</h1>"     > /var/www/intranet/index.html
```

**`/etc/apache2/sites-available/www.conf`:**

```apache
<VirtualHost *:80>
    ServerName www.g07rc2.com.br
    Redirect permanent / https://www.g07rc2.com.br/
    ErrorLog ${APACHE_LOG_DIR}/www_error.log
    CustomLog ${APACHE_LOG_DIR}/www_access.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName www.g07rc2.com.br
    DocumentRoot /var/www/www
    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/web.crt
    SSLCertificateKeyFile /etc/ssl/private/web.key
    ErrorLog ${APACHE_LOG_DIR}/www_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/www_ssl_access.log combined
</VirtualHost>
```

**`/etc/apache2/sites-available/intranet.conf`:**

```apache
<VirtualHost *:80>
    ServerName intranet.g07rc2.com.br
    Redirect permanent / https://intranet.g07rc2.com.br/
    ErrorLog ${APACHE_LOG_DIR}/intranet_error.log
    CustomLog ${APACHE_LOG_DIR}/intranet_access.log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName intranet.g07rc2.com.br
    DocumentRoot /var/www/intranet
    SSLEngine on
    SSLCertificateFile    /etc/ssl/certs/web.crt
    SSLCertificateKeyFile /etc/ssl/private/web.key
    ErrorLog ${APACHE_LOG_DIR}/intranet_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/intranet_ssl_access.log combined

    <Directory /var/www/intranet>
        AuthType Basic
        AuthName "Intranet g07rc2"
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user
    </Directory>
</VirtualHost>
```

Usuário da intranet e ativação:

```bash
htpasswd -c /etc/apache2/.htpasswd admin
a2dissite 000-default
a2ensite www intranet
apache2ctl configtest
service apache2 restart
systemctl enable apache2
```

Logs separados por site: `/var/log/apache2/` (`www_*.log` e `intranet_*.log`). Falhas para investigar na defesa: acessar a intranet sem credenciais (401 em `intranet_ssl_access.log`) ou errar o caminho (404).

> **➜ Reflexo em outras máquinas:** cada `ServerName` precisa existir no DNS (1.6) e constar no `subjectAltName` do certificado (1.8). No **[CLIENTE]**, o navegador só mostra o cadeado depois de importar a CA (3.3). Para mudar o nome de um site: DNS (com serial maior), `ServerName`, certificado refeito e `service apache2 restart`.

---

## 2. Servidor de e-mail **[EXTRA]** **[E-MAIL]**

Nessa máquina rodam **só** Postfix (SMTP) e Dovecot (IMAP). Nada de DHCP, DNS ou Apache.

> **Antes de começar, precisa estar pronto na infra:** NAT (1.4, para o `apt-get`), DNS com os registros `mail` e MX (1.6) e a CA com `mail.crt` e `mail.key` (1.8).

### 2.1 Nome e rede (IP fixo), no estilo dos slides

Confirme o nome da interface (`ip a`; abaixo uso `enp0s3`). Edite `/etc/network/interfaces`:

```
auto lo
iface lo inet loopback

allow-hotplug enp0s3
iface enp0s3 inet static
address 172.16.0.2
netmask 255.255.0.0
gateway 172.16.0.1
```

DNS no `/etc/resolv.conf`:

```
search g07rc2.com.br
nameserver 172.16.0.1
```

**Nome da máquina.** Esta máquina passa a se chamar `mail`:

```bash
hostnamectl set-hostname mail
```

Em `/etc/hosts`, mantenha `127.0.0.1 localhost` e substitua a linha `127.0.1.1` que já existe por:

```
127.0.1.1   mail.g07rc2.com.br mail
```

Depois reinicie:

```bash
reboot
```

**Evidência:** `hostname` (mostra `mail`), `hostname -f` (mostra `mail.g07rc2.com.br`), `ip a`, `ping 172.16.0.1` e `ping 8.8.8.8` (se o segundo funcionar, o NAT do servidor de infra está ok).

> **➜ Reflexo em outras máquinas:** o nome `mail` precisa existir no DNS da **infra** (registro A e MX, em 1.6). Ele é o mesmo do `myhostname` do Postfix (2.4) e do certificado `mail.crt` (1.8). O IP `172.16.0.2` precisa ser o mesmo do registro A, do PTR `2.0` e dos comandos `ssh` e `ssh-copy-id` do **[CLIENTE]**.

### 2.2 Preparação e instalação

Faça a mesma preparação do item 1.1 (PATH do root, `net-tools`, `vim`, `rsyslog`; o hostname você já definiu em 2.1) e instale. O `ssh` entra agora porque a infra vai enviar o certificado por `scp` no próximo passo:

```bash
apt-get update
apt-get install postfix dovecot-imapd rsyslog ssh
adduser aluno
```

No Postfix, escolha **Internet Site** e `g07rc2.com.br`. O `rsyslog` cria o `/var/log/mail.log`. O usuário `aluno` é o que recebe o `scp` e o `ssh-copy-id` (o root não entra por SSH).

### 2.3 Certificado (sai da infra e chega no e-mail)

No servidor de infra:

```bash
scp ~/ca/mail.crt ~/ca/mail.key aluno@172.16.0.2:/tmp/
```

No servidor de e-mail:

```bash
mv /tmp/mail.crt /etc/ssl/certs/mail.crt
mv /tmp/mail.key /etc/ssl/private/mail.key
chmod 640 /etc/ssl/private/mail.key
chgrp ssl-cert /etc/ssl/private/mail.key
```

### 2.4 Postfix

Faça backup e edite `/etc/postfix/main.cf` (ajuste ou acrescente):

```bash
cp /etc/postfix/main.cf /etc/postfix/main.cf.bkp
```

```
myhostname = mail.g07rc2.com.br
mydomain = g07rc2.com.br
myorigin = $mydomain
mydestination = $myhostname, $mydomain, localhost
inet_interfaces = all
mynetworks = 127.0.0.0/8
home_mailbox = Maildir/

smtpd_tls_cert_file = /etc/ssl/certs/mail.crt
smtpd_tls_key_file = /etc/ssl/private/mail.key
smtpd_tls_security_level = may
smtpd_tls_auth_only = yes

smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_recipient_restrictions = permit_sasl_authenticated, reject_unauth_destination
```

Em `/etc/postfix/master.cf`, descomente e ajuste o bloco `submission`:

```
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
```

### 2.5 Dovecot

- `/etc/dovecot/dovecot.conf`: `protocols = imap`
- `/etc/dovecot/conf.d/10-mail.conf`: `mail_location = maildir:~/Maildir`
- `/etc/dovecot/conf.d/10-auth.conf`: `disable_plaintext_auth = yes` e `auth_mechanisms = plain login`
- `/etc/dovecot/conf.d/10-ssl.conf`:

  ```
  ssl = required
  ssl_cert = </etc/ssl/certs/mail.crt
  ssl_key = </etc/ssl/private/mail.key
  ```

- `/etc/dovecot/conf.d/10-master.conf`, dentro de `service auth { ... }`:

  ```
  unix_listener /var/spool/postfix/private/auth {
      mode = 0660
      user = postfix
      group = postfix
  }
  ```

### 2.6 Contas, serviços e SSH

```bash
adduser ana
adduser bia
service postfix restart
service dovecot restart
systemctl enable postfix dovecot
```

SSH: o `ssh` já foi instalado em 2.2. Falta copiar a chave do cliente (`ssh-copy-id aluno@172.16.0.2`, rodado no **[CLIENTE]**) e, só depois de testar, desativar a senha aqui, como em 1.5.

### 2.7 Verificações

```bash
ss -tlnp | grep -E ':25|:587|:993'
tail -f /var/log/mail.log
```

> **➜ Reflexo em outras máquinas:** com Postfix e Dovecot no ar, o **[CLIENTE]** precisa ter a CA importada no Thunderbird (3.3) para aceitar o certificado. A conta usa o nome `mail.g07rc2.com.br` (resolvido pelo DNS da infra) e o MX da zona precisa apontar para esse nome. Se mudar o `myhostname`, ajuste também o DNS, o certificado (se o nome do CN/SAN mudar) e a conta do Thunderbird.

---

## 3. Estação cliente (Debian com interface) **[CLIENTE]**

### 3.1 Rede via DHCP

O cliente mantém o nome que já tem: o trabalho não pede renomear nem registrar o cliente no DNS. Ele só precisa receber tudo pelo DHCP.

Deixe a conexão cabeada em **Automático (DHCP)**. Confira no terminal:

```bash
ip a                       # IP recebido (172.16.1.x)
ip r                       # default via 172.16.0.1
cat /etc/resolv.conf       # nameserver 172.16.0.1 e search g07rc2.com.br
```

Renovar a concessão: `nmcli connection down "Conexão cabeada 1" && nmcli connection up "Conexão cabeada 1"`.

> **➜ Reflexo em outras máquinas:** o cliente só recebe IP se a **infra** estiver ligada com o DHCP (1.3) no ar, e só instala pacotes se o NAT (1.4) estiver funcionando. Se o cliente não pegar IP, confira também se a Rede interna das VMs tem o mesmo nome.

### 3.2 Ferramentas

```bash
apt-get install dnsutils smbclient cifs-utils thunderbird wireshark curl
```

### 3.3 Instalar a CA no cliente **[EXTRA]**

```bash
scp aluno@172.16.0.1:ca.crt ~/
cp ~/ca.crt /usr/local/share/ca-certificates/ca-g07rc2.crt
update-ca-certificates
```

> O Firefox e o Thunderbird usam **repositório próprio** de certificados. Importe a CA neles:
> *Configurações → Privacidade e Segurança → Certificados → Ver Certificados → Autoridades → Importar*, e marque "Confiar nesta CA para identificar sites".
> Sem isso, o HTTPS e o IMAP aparecem como inválidos na demonstração.

### 3.4 Testes

```bash
# DNS
dig www.g07rc2.com.br
dig g07rc2.com.br MX
dig -x 172.16.0.2

# SSH
ssh aluno@172.16.0.1

# Samba
smbclient //172.16.0.1/Dados_Publicos -U aluno

# Web (validação do certificado)
curl -v https://www.g07rc2.com.br
curl -v -u admin https://intranet.g07rc2.com.br
```

### 3.5 E-mail (Thunderbird)

- Conta `ana@g07rc2.com.br`
- **IMAP**: `mail.g07rc2.com.br`, porta `993`, SSL/TLS, usuário `ana`
- **SMTP**: `mail.g07rc2.com.br`, porta `587`, STARTTLS, usuário `ana`
- Envie para `bia@g07rc2.com.br`; depois abra a conta da `bia` e leia a mensagem.

---

## 4. Capturas de pacotes comentadas

No servidor de infra, grave enquanto o cliente faz a ação:

```bash
tcpdump -i enp0s8 -w dhcp.pcap port 67 or port 68
tcpdump -i enp0s8 -w dns.pcap port 53
tcpdump -i enp0s8 -w tls.pcap port 443
```

Abra no Wireshark e **identifique as etapas** no relatório:

- **DHCP**: `DHCPDISCOVER` → `DHCPOFFER` → `DHCPREQUEST` → `DHCPACK` (é o mesmo ciclo do slide 7 da aula 3).
- **DNS**: Query → Response (tipos A, MX, PTR e o NXDOMAIN).
- **TLS**: ClientHello → ServerHello + Certificate → troca de chaves → dados cifrados.
- **E-mail**: conexão ao 587, EHLO, STARTTLS, AUTH, MAIL FROM, RCPT TO, DATA.

---

## 5. Roteiro da demonstração (nessa ordem)

1. Cliente **sem IP manual**: mostrar `ip a`, `ip r`, `resolv.conf` (DHCP, gateway e DNS).
2. `dig` dos nomes web e do MX.
3. SSH por chave e teste das permissões do Samba.
4. Os dois sites no navegador (autenticação na intranet, cadeado do HTTPS válido e redirecionamento do HTTP).
5. Envio e leitura de e-mail pelo cliente, mostrando `/var/log/mail.log` no servidor.

**Teste de reinicialização:** reinicie as três máquinas antes da apresentação e repita os passos 1 a 5. Ligue nesta ordem: **infra** primeiro (NAT, DHCP e DNS), depois o **e-mail** e por último o **cliente**.

```bash
systemctl is-enabled isc-dhcp-server bind9 apache2 ssh smbd nmbd     # infra
systemctl is-enabled postfix dovecot ssh                              # e-mail
iptables -t nat -L -n -v                                              # NAT após o reboot
cat /etc/iptables/rules.v4
```

---

## 6. Diagnóstico rápido

| Problema | Onde olhar |
|---|---|
| Cliente não pega IP | `service isc-dhcp-server status`, `INTERFACESv4`, rede interna igual nas VMs |
| Ping falha | IP, interface, estado da VM, firewall |
| Sem internet no cliente/e-mail | `ip_forward`, regra NAT, interface `enp0s3` |
| Nome não resolve | `named-checkzone`, `journalctl -u bind9 -n 40`, `dig @172.16.0.1 ...` |
| Site não abre | `apache2ctl configtest`, `/var/log/apache2/*_error.log` |
| Certificado não confiável | CA importada no navegador? `subjectAltName` bate com o nome do site? |
| Samba nega acesso | `testparm`, `smbpasswd -a`, grupo do usuário, `/var/log/samba/` |
| E-mail não autentica | `/var/log/mail.log`, socket `private/auth` do Dovecot, porta 587 |
| `command not found` | pacote ausente ou PATH do root incompleto (item 1.1) |

---

## 7. Repositório

Estrutura sugerida:

```
repo/
├── README.md              (instruções para reconstruir o ambiente)
├── diagrama.png           (topologia)
├── plano-enderecamento.md
├── infra/
│   ├── interfaces
│   ├── hosts
│   ├── dhcpd.conf
│   ├── isc-dhcp-server
│   ├── named.conf.local
│   ├── named.conf.options
│   ├── db.g07rc2.com.br
│   ├── db.172.16.rev
│   ├── rules.v4
│   ├── smb.conf
│   ├── sshd_config
│   └── apache/ (www.conf, intranet.conf)
├── mail/
│   ├── interfaces
│   ├── hosts
│   ├── main.cf
│   ├── master.cf
│   └── dovecot/ (arquivos alterados)
├── ca/
│   └── ca.crt             (só o certificado público!)
├── capturas/              (.pcap + comentários)
└── relatorio.pdf
```

**Não enviar:** `*.key`, `ca.key`, `.htpasswd`, o arquivo `.samba` (credenciais), senhas e chaves SSH privadas.

Prazo de entrega: **09/12/2026** (não serão aceitos trabalhos em atraso).
