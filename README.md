# Trabalho Final — Redes de Computadores 2

Instituto Federal Sul-Rio-Grandense, Campus Passo Fundo
Curso: Bacharelado em Ciência da Computação · Prof.: Roberto Wiest
Integrantes: Matheus Witte Ditz, Andrey DC Silva

Infraestrutura de uma organização fictícia (domínio `g07rc2.com.br`): roteamento/NAT, DHCP, DNS, SSH, Samba, Apache (HTTP/HTTPS) e e-mail, tudo persistente após reinicialização.

> Este repositório **não contém** senhas, chaves privadas (`*.key`), `.htpasswd` nem arquivos de credenciais. A seção 5 explica como gerá-los.

---

## 1. Visão geral

Diagrama da topologia: [`diagrama.png`](diagrama.png)
Plano de endereçamento: [`plano-enderecamento.md`](plano-enderecamento.md)

| Máquina | Hostname | Interface | IP | Serviços |
|---|---|---|---|---|
| Servidor de infraestrutura | `infra.g07rc2.com.br` | `enp0s3` (externa/NAT) | DHCP da VM | NAT, Samba, SSH |
| | | `enp0s8` (interna) | `172.16.0.1/16` | DHCP, DNS (BIND9), Apache, Samba, SSH |
| Servidor de e-mail | `mail.g07rc2.com.br` | `<PREENCHER>` | `172.16.0.2/16` | Postfix (SMTP), Dovecot (IMAP), SSH |
| Estação cliente | `cliente` | `<PREENCHER>` | DHCP (`172.16.1.100–172.16.1.200`) | Navegador, cliente de arquivos, Thunderbird |

- Rede interna: `172.16.0.0/16` (Classe B) · Gateway: `172.16.0.1` · DNS: `172.16.0.1`
- Reserva por MAC: `cliente` → `172.16.1.50` (MAC `<PREENCHER>`)
- Domínio de busca entregue por DHCP: `g07rc2.com.br`

---

## 2. Pré-requisitos

- Debian `<PREENCHER: versão>`: servidores em modo terminal (sem interface gráfica) e cliente com interface gráfica.
- VMs (`<PREENCHER: VirtualBox/outro>`):
  - **Infra**: 2 placas (Adaptador 1 = NAT, Adaptador 2 = Rede interna `<PREENCHER: nome>`).
  - **E-mail** e **Cliente**: 1 placa cada, na mesma Rede interna.
- Acesso à internet pelo NAT da VM de infra (necessário para o `apt-get`).
- Pacotes por máquina:

| Máquina | Pacotes |
|---|---|
| Infra | `net-tools iptables iptables-persistent vim rsyslog isc-dhcp-server bind9 dnsutils ssh samba smbclient cifs-utils apache2 apache2-utils openssl` |
| E-mail | `net-tools vim rsyslog ssh postfix dovecot-imapd` |
| Cliente | `dnsutils smbclient cifs-utils thunderbird wireshark curl` |

---

## 3. Mapa dos arquivos do repositório

| No repositório | Destino na máquina |
|---|---|
| `infra/interfaces` | `/etc/network/interfaces` |
| `infra/isc-dhcp-server` | `/etc/default/isc-dhcp-server` |
| `infra/dhcpd.conf` | `/etc/dhcp/dhcpd.conf` |
| `infra/rules.v4` | `/etc/iptables/rules.v4` |
| `infra/named.conf.options` | `/etc/bind/named.conf.options` |
| `infra/named.conf.local` | `/etc/bind/named.conf.local` |
| `infra/db.g07rc2.com.br` | `/etc/bind/db.g07rc2.com.br` |
| `infra/db.172.16.rev` | `/etc/bind/db.172.16.rev` |
| `infra/sshd_config` | `/etc/ssh/sshd_config` |
| `infra/smb.conf` | `/etc/samba/smb.conf` |
| `infra/apache/www.conf` | `/etc/apache2/sites-available/www.conf` |
| `infra/apache/intranet.conf` | `/etc/apache2/sites-available/intranet.conf` |
| `mail/interfaces` | `/etc/network/interfaces` (máquina de e-mail) |
| `mail/main.cf`, `mail/master.cf` | `/etc/postfix/` |
| `mail/dovecot/*` | `/etc/dovecot/` e `/etc/dovecot/conf.d/` |
| `ca/ca.crt` | certificado **público** da CA (instalar no cliente) |

---

## 4. Como reconstruir o ambiente

Execute como root. Siga a ordem: o servidor de e-mail só tem internet depois que o NAT da infra estiver pronto.

### 4.1 Servidor de infraestrutura

1. **Preparação**
   ```bash
   apt-get update
   apt-get install net-tools iptables iptables-persistent vim rsyslog
   echo 'export PATH=$PATH:/sbin:/bin:/usr/sbin' >> /root/.bashrc
   hostnamectl set-hostname infra.g07rc2.com.br
   ```
   Em `/etc/hosts`, adicione `127.0.1.1   infra.g07rc2.com.br infra`.

2. **Rede:** copie `infra/interfaces` para `/etc/network/interfaces` e reinicie (`reboot`).
   Verificação: `ip a` (enp0s3 dinâmico, enp0s8 com `172.16.0.1/16`).

3. **Roteamento e NAT**
   - Em `/etc/sysctl.conf`, deixe `net.ipv4.ip_forward=1` e rode `sysctl -p`.
   - Regra de NAT:
     ```bash
     iptables -t nat -A POSTROUTING -o enp0s3 -s 172.16.0.0/16 -j MASQUERADE
     iptables-save > /etc/iptables/rules.v4
     ```
     (ou copie `infra/rules.v4` para `/etc/iptables/rules.v4`).
   - Verificação após `reboot`: `cat /etc/iptables/rules.v4`.

4. **DHCP**
   ```bash
   apt-get install isc-dhcp-server
   cp infra/isc-dhcp-server /etc/default/isc-dhcp-server
   mv /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.original
   cp infra/dhcpd.conf /etc/dhcp/dhcpd.conf
   service isc-dhcp-server restart
   systemctl enable isc-dhcp-server
   ```
   Ajuste o MAC da reserva em `dhcpd.conf` para o MAC real do cliente.

5. **DNS (BIND9)**
   ```bash
   apt-get install bind9 dnsutils
   cp infra/named.conf.options /etc/bind/
   cp infra/named.conf.local   /etc/bind/
   cp infra/db.g07rc2.com.br   /etc/bind/
   cp infra/db.172.16.rev      /etc/bind/
   named-checkconf
   named-checkzone g07rc2.com.br /etc/bind/db.g07rc2.com.br
   named-checkzone 16.172.in-addr.arpa /etc/bind/db.172.16.rev
   service bind9 restart
   systemctl enable bind9
   ```
   Em `/etc/resolv.conf` da infra: `search g07rc2.com.br` e `nameserver 127.0.0.1`.

6. **SSH com chave**
   ```bash
   apt-get install ssh
   cp infra/sshd_config /etc/ssh/sshd_config
   sshd -t && service ssh restart
   ```
   A chave pública do cliente precisa estar em `~/.ssh/authorized_keys` (ver seção 5).

7. **Samba**
   ```bash
   apt-get install samba smbclient cifs-utils
   cp /etc/samba/smb.conf /etc/samba/smb.conf.bkp
   cp infra/smb.conf /etc/samba/smb.conf
   mkdir -p /mnt/Publico /mnt/grupos /mnt/usuario
   groupadd ifsulalunos
   # criar usuários e senhas: ver seção 5
   testparm
   service smbd restart && service nmbd restart
   systemctl enable smbd nmbd
   ```
   Permissões das pastas: ver `infra/smb.conf` e a seção 5.

8. **CA local e certificados:** gerar conforme a seção 5.1 e instalar `web.crt`/`web.key` em `/etc/ssl/certs/` e `/etc/ssl/private/`.

9. **Apache**
   ```bash
   apt-get install apache2 apache2-utils
   a2enmod ssl rewrite headers
   mkdir -p /var/www/www /var/www/intranet
   cp infra/apache/*.conf /etc/apache2/sites-available/
   a2dissite 000-default
   a2ensite www intranet
   apache2ctl configtest
   service apache2 restart
   systemctl enable apache2
   ```
   Os conteúdos dos sites estão em `infra/www/` e `infra/intranet/` (`<PREENCHER, se incluir no repositório>`).

### 4.2 Servidor de e-mail

1. **Preparação e rede:** mesmos passos do item 4.1.1; copie `mail/interfaces` e ajuste `/etc/resolv.conf` (`search g07rc2.com.br`, `nameserver 172.16.0.1`). Hostname: `mail.g07rc2.com.br`. Reinicie.
2. **Pacotes:** `apt-get install postfix dovecot-imapd rsyslog` (Postfix: "Internet Site", `g07rc2.com.br`).
3. **Certificado do e-mail:** gerar na CA (seção 5.1) e copiar para `/etc/ssl/certs/mail.crt` e `/etc/ssl/private/mail.key`.
4. **Configuração:** copie `mail/main.cf` e `mail/master.cf` para `/etc/postfix/`, e os arquivos de `mail/dovecot/` para `/etc/dovecot/`.
5. **Contas e serviços**
   ```bash
   adduser ana
   adduser bia
   service postfix restart
   service dovecot restart
   systemctl enable postfix dovecot
   ```
6. **SSH:** mesmo procedimento do item 4.1.6.

### 4.3 Estação cliente

1. Interface cabeada em **DHCP** (sem IP manual).
2. Instale os pacotes da seção 2.
3. Instale a CA:
   ```bash
   cp ca/ca.crt /usr/local/share/ca-certificates/ca-g07rc2.crt
   update-ca-certificates
   ```
   Importe também `ca.crt` no Firefox e no Thunderbird (Configurações → Privacidade e Segurança → Certificados → Autoridades), porque eles têm repositório próprio.
4. Configure o Thunderbird: IMAP `mail.g07rc2.com.br:993` (SSL/TLS) e SMTP `mail.g07rc2.com.br:587` (STARTTLS).
5. (Opcional) Montagem automática do Samba em `/etc/fstab`:
   ```
   //172.16.0.1/homes   /mnt/aluno   cifs   defaults,credentials=/home/aluno/.samba   0   0
   ```

---

## 5. O que NÃO está no repositório (como gerar)

**5.1 CA e certificados** (no servidor de infra, em `~/ca`)
```bash
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/C=BR/O=g07rc2/CN=CA Laboratorio g07rc2" -out ca.crt
```
Para cada certificado (`web` com `subjectAltName = DNS:www.g07rc2.com.br, DNS:intranet.g07rc2.com.br`; `mail` com `DNS:mail.g07rc2.com.br`): gerar chave, CSR e assinar com `openssl x509 -req ... -CA ca.crt -CAkey ca.key -extfile <arquivo.ext>`.

**5.2 Chaves SSH** (no cliente)
```bash
ssh-keygen -t ed25519
ssh-copy-id usuario@172.16.0.1
ssh-copy-id usuario@172.16.0.2
```

**5.3 Senhas e contas**
```bash
htpasswd -c /etc/apache2/.htpasswd admin     # intranet
adduser aluno && smbpasswd -a aluno           # Samba
adduser professor && smbpasswd -a professor
gpasswd -a aluno ifsulalunos
```
Arquivo de credenciais do Samba no cliente (`/home/aluno/.samba`, com `chmod 600`): `user=`, `password=`, `domain=RC2`.

Defina as senhas de forma privada e informe ao professor na demonstração, se necessário.

---

## 6. Como validar

Na ordem da demonstração (cliente sem IP manual):

1. `ip a`, `ip r`, `cat /etc/resolv.conf`: IP por DHCP, gateway `172.16.0.1`, DNS `172.16.0.1`.
2. `dig www.g07rc2.com.br`, `dig intranet.g07rc2.com.br`, `dig g07rc2.com.br MX`, `dig -x 172.16.0.2`, `dig naoexiste.g07rc2.com.br` (NXDOMAIN).
3. `ssh usuario@172.16.0.1` (por chave) e `smbclient //172.16.0.1/diretorio_grupos -U aluno` (permitido) / `-U professor` (negado).
4. Navegador: `http://www.g07rc2.com.br` redireciona para HTTPS; `https://intranet.g07rc2.com.br` pede autenticação; certificado válido.
5. Thunderbird: enviar de `ana@g07rc2.com.br` para `bia@g07rc2.com.br` e ler por IMAP; logs em `/var/log/mail.log` no servidor de e-mail.

**Persistência:** reinicie as três máquinas e repita os testes. Conferir também `systemctl is-enabled ...` e `cat /etc/iptables/rules.v4`.

---

## 7. Capturas de pacotes

Em [`capturas/`](capturas/): `<PREENCHER: nomes dos arquivos .pcap>` e os comentários explicando as etapas do protocolo (DHCP: Discover/Offer/Request/Ack; DNS: query/response; TLS: handshake).

## 8. Relatório técnico

[`relatorio.pdf`](relatorio.pdf): decisões, problemas encontrados e correções.
