# Guia do Trabalho Final — Redes de Computadores 2

Domínio: `g07rc2.com.br` · Debian (servidores em modo terminal, cliente com interface gráfica)

Este guia segue o **passo a passo dos slides do professor** (aulas 2 a 6). Onde o trabalho exige algo diferente dos slides, a seção avisa com a marcação **[ADAPTADO]**. O que não existe nos slides (HTTPS, CA, e-mail, SSH por chave) vem marcado como **[EXTRA]**.

> Convenções dos slides: todos os comandos de servidor rodam como **root** (por isso o ajuste do PATH da seção 1.1). Reinicie serviços com `service <nome> restart`. Cada etapa termina com uma **Evidência**: é o que o professor quer ver (e o que você deve copiar para o relatório).

---

## 0. O que muda em relação aos slides

| Slides | Trabalho | O que fazer |
|---|---|---|
| Rede `192.168.10.0/24` (Classe C) | O trabalho exige rede interna **Classe B** | Usar `172.16.0.0/16` (máscara `255.255.0.0`) no lugar |
| Domínio `disciplinacsr.com.br` | Domínio do grupo: `g07rc2.com.br` | Trocar em DNS, Apache e e-mail |
| Zona reversa `10.168.192.in-addr.arpa` (/24) | Rede /16 | Zona reversa `16.172.in-addr.arpa` e PTR com **dois octetos** (`1.0`, `2.0`) |
| DHCP entrega IP, gateway e DNS | O trabalho também pede **domínio de busca** | Acrescentar `option domain-name` e `option domain-search` |
| SSH com senha | Trabalho pede **autenticação por chave** | `ssh-keygen` + `ssh-copy-id` e desativar senha **[EXTRA]** |
| Não há | Apache, HTTPS, CA local, e-mail | Seções 1.9, 1.10 e 2 **[EXTRA]** |

### Plano de endereçamento

| Máquina | Hostname | Interface | IP |
|---|---|---|---|
| Servidor de infraestrutura | `infra.g07rc2.com.br` | `enp0s3` (externa/NAT da VM) | DHCP da VM |
| | | `enp0s8` (interna) | `172.16.0.1/16` (fixo) |
| Servidor de e-mail | `mail.g07rc2.com.br` | interna | `172.16.0.2/16` (fixo) |
| Estação cliente | `cliente` | interna | DHCP, faixa `172.16.1.100–172.16.1.200` |

- Gateway da rede interna: `172.16.0.1` · DNS: `172.16.0.1` · Broadcast: `172.16.255.255`
- Reserva por MAC: o cliente fica com `172.16.1.50`.

### VMs (VirtualBox ou similar)

- **Infra**: 2 placas. Adaptador 1 = NAT. Adaptador 2 = Rede interna (ex.: `redeint`).
- **E-mail** e **Cliente**: 1 placa cada, na mesma Rede interna `redeint`.

### Ordem recomendada

1. Infra: preparação, rede, NAT (o servidor de e-mail precisa de internet para o `apt-get`).
2. Infra: DHCP, DNS.
3. Infra: SSH, Samba, CA, Apache.
4. E-mail: rede, Postfix, Dovecot.
5. Cliente: DHCP, CA, testes.
6. Capturas, repositório, relatório.

---

## 1. Servidor de infraestrutura

### 1.1 Preparação do ambiente (Aula 2)

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

Defina o hostname e o `/etc/hosts`:

```bash
hostnamectl set-hostname infra.g07rc2.com.br
```

Em `/etc/hosts`:

```
127.0.1.1   infra.g07rc2.com.br infra
```

**Evidência:** `hostname`, `whoami`, `pwd`.

### 1.2 Configuração de rede (Aula 2) **[ADAPTADO: Classe B]**

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

### 1.3 DHCP (Aula 3) **[ADAPTADO: Classe B + domínio de busca]**

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

### 1.4 Roteamento, NAT e persistência (Aula 3)

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

### 1.5 SSH (Aula 4)

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
ssh-copy-id aluno@172.16.0.2     # depois de configurar o servidor de e-mail
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

### 1.6 DNS com BIND9 (Aula 6) **[ADAPTADO: domínio e Classe B]**

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

### 1.7 Samba (Aula 5)

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

### 1.8 Autoridade certificadora (CA) local **[EXTRA]**

No servidor de infra, em `~/ca`:

```bash
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

> **Nunca** suba `*.key` (privadas) para o repositório. Use `.gitignore` para `*.key` e `.htpasswd`.

### 1.9 Apache com dois sites (hosts virtuais) **[EXTRA]**

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

---

## 2. Servidor de e-mail **[EXTRA]**

Nessa máquina rodam **só** Postfix (SMTP) e Dovecot (IMAP). Nada de DHCP, DNS ou Apache.

### 2.1 Rede (IP fixo), no estilo dos slides

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

```bash
hostnamectl set-hostname mail.g07rc2.com.br
reboot
```

Em `/etc/hosts`: `127.0.1.1   mail.g07rc2.com.br mail`.

**Evidência:** `ip a`, `ping 172.16.0.1` e `ping 8.8.8.8` (se o segundo funcionar, o NAT do servidor de infra está ok).

### 2.2 Preparação e instalação

Faça a mesma preparação do item 1.1 (PATH do root, `net-tools`, `vim`, `rsyslog`) e instale:

```bash
apt-get update
apt-get install postfix dovecot-imapd rsyslog
```

No Postfix, escolha **Internet Site** e `g07rc2.com.br`. O `rsyslog` cria o `/var/log/mail.log`.

### 2.3 Certificado

No servidor de infra:

```bash
scp ~/ca/mail.crt ~/ca/mail.key usuario@172.16.0.2:/tmp/
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

SSH: mesmo procedimento do item 1.5 (`apt-get install ssh`, copiar a chave do cliente, desativar senha).

### 2.7 Verificações

```bash
ss -tlnp | grep -E ':25|:587|:993'
tail -f /var/log/mail.log
```

---

## 3. Estação cliente (Debian com interface)

### 3.1 Rede via DHCP

Deixe a conexão cabeada em **Automático (DHCP)**. Confira no terminal:

```bash
ip a                       # IP recebido (172.16.1.x)
ip r                       # default via 172.16.0.1
cat /etc/resolv.conf       # nameserver 172.16.0.1 e search g07rc2.com.br
```

Renovar a concessão: `nmcli connection down "Conexão cabeada 1" && nmcli connection up "Conexão cabeada 1"`.

### 3.2 Ferramentas

```bash
apt-get install dnsutils smbclient cifs-utils thunderbird wireshark curl
```

### 3.3 Instalar a CA no cliente **[EXTRA]**

```bash
scp usuario@172.16.0.1:~/ca/ca.crt ~/
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

**Teste de reinicialização:** reinicie as três máquinas antes da apresentação e repita os passos 1 a 5.

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
