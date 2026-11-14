# Trabalho Final — Redes de Computadores 2 (Grupo 07)

Instituto Federal Sul-Rio-Grandense, Campus Passo Fundo
Curso: Bacharelado em Ciência da Computação · Prof.: Roberto Wiest
Integrantes: `<PREENCHER: nomes>`

Infraestrutura de uma organização fictícia (domínio `g07rc2.com.br`): roteamento/NAT, DHCP, DNS, SSH, Samba, Apache (HTTP/HTTPS) e e-mail, tudo persistente após reinicialização.

> Este repositório **não contém** senhas, chaves privadas (`*.key`), `.htpasswd` nem arquivos de credenciais. A seção 5 explica como gerá-los.

---

## 1. Visão geral

Diagrama da topologia: [`topologia.jpg`](topologia.jpg)
Plano de endereçamento: [`guia_trabalho_redes2 atualizado.md`](guia_trabalho_redes2 atualizado.md)

| Máquina | Hostname | Interface | IP | Serviços |
|---|---|---|---|---|
| Servidor de infraestrutura | `infra` | `enp0s3` (externa/NAT) | DHCP da VM | NAT, Samba, SSH |
| | | `enp0s8` (interna) | `172.16.0.1/16` | DHCP, DNS (BIND9), Apache, Samba, SSH |
| Servidor de e-mail | `mail` | `<PREENCHER>` | `172.16.0.2/16` | Postfix (SMTP), Dovecot (IMAP), SSH |
| Estação cliente | nome padrão (não alterado) | `<PREENCHER>` | DHCP (`172.16.1.100–172.16.1.200`) | Navegador, cliente de arquivos, Thunderbird |

- **Nomes:** os servidores usam hostname curto (`infra`, `mail`). O nome completo (`infra.g07rc2.com.br`, `mail.g07rc2.com.br`) vem do `/etc/hosts` de cada máquina e dos registros do DNS. O cliente mantém o nome padrão e não tem registro no DNS.
- Rede interna: `172.16.0.0/16` (Classe B) · Gateway: `172.16.0.1` · DNS: `172.16.0.1`
- Reserva por MAC: cliente → `172.16.1.50` (MAC `<PREENCHER>`). O `host cliente` do `dhcpd.conf` é só um rótulo: o que vale é o MAC.
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
| Infra | `net-tools iptables iptables-persistent vim rsyslog isc-dhcp-server bind9 dnsutils ssh samba smbclient cifs-utils apache2 apache2-utils openssl ssl-cert` |
| E-mail | `net-tools vim rsyslog ssh postfix dovecot-imapd` |
| Cliente | `dnsutils smbclient cifs-utils thunderbird wireshark curl` |

---

## 3. Mapa dos arquivos do repositório

| No repositório | Destino na máquina |
|---|---|
| `infra/hosts` | `/etc/hosts` (referência: a linha `127.0.1.1`) |
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
| `mail/hosts` | `/etc/hosts` (máquina de e-mail; referência: a linha `127.0.1.1`) |
| `mail/interfaces` | `/etc/network/interfaces` (máquina de e-mail) |
| `mail/main.cf`, `mail/master.cf` | `/etc/postfix/` |
| `mail/dovecot/*` | `/etc/dovecot/` e `/etc/dovecot/conf.d/` |
| `ca/ca.crt` | certificado **público** da CA (instalar no cliente) |

---

## 4. Como reconstruir o ambiente

Execute como root. Siga a ordem abaixo: cada etapa depende da anterior.

| # | Máquina | Itens | Precisa estar pronto antes |
|---|---|---|---|
| 1 | Infra | 4.1.1 a 4.1.3 (nome, rede, NAT) | VM com 2 placas |
| 2 | Infra | 4.1.4 e 4.1.5 (DHCP, DNS) | NAT funcionando |
| 3 | Cliente | 4.3.1 e 4.3.2 (DHCP, pacotes) | DHCP e NAT da infra |
| 4 | Infra | 4.1.6 e 4.1.7 (SSH, Samba) | Cliente com IP |
| 5 | Infra | 4.1.8 e 4.1.9 (CA, Apache) | Nomes `www` e `intranet` no DNS |
| 6 | E-mail | 4.2.1 e 4.2.2 (nome, rede, pacotes, `ssh`) | NAT e DNS da infra |
| 7 | Infra e E-mail | 4.2.3 a 4.2.6 (certificado, Postfix, Dovecot) | CA gerada (4.1.8) e `ssh` instalado no e-mail |
| 8 | Cliente | 4.3.3 a 4.3.5 (CA, Thunderbird) | Etapas 5 e 7 |

Se você mudar algo no meio do caminho, confira também:

| Se mudar... | Confira também... |
|---|---|
| IP da infra | `routers` e `domain-name-servers` do DHCP, registros A e PTR do DNS, `gateway` e `nameserver` do e-mail, comandos do cliente |
| IP do e-mail | Registro A e PTR do DNS, `ssh-copy-id` e `ssh` do cliente |
| Nome de um serviço (`www`, `intranet`, `mail`) | Zona DNS (aumentar o serial), `ServerName` do Apache, `subjectAltName` do certificado (refazer e reinstalar), conta do Thunderbird |
| Zona do DNS | Serial maior, `named-checkzone`, `service bind9 restart` |
| Nome de interface (`enp0s3`, `enp0s8`) | `interfaces`, `isc-dhcp-server`, regra NAT (`-o`), `tcpdump -i` |
| CA ou certificados | Reinstalar no Apache, copiar `mail.crt` e `mail.key` para o e-mail, reimportar a CA no cliente, reiniciar Apache, Postfix e Dovecot |

### 4.1 Servidor de infraestrutura

1. **Preparação e nome**
   ```bash
   apt-get update
   apt-get install net-tools iptables iptables-persistent vim rsyslog
   echo 'export PATH=$PATH:/sbin:/bin:/usr/sbin' >> /root/.bashrc
   hostnamectl set-hostname infra
   ```
   Em `/etc/hosts`, mantenha `127.0.0.1 localhost` e substitua a linha `127.0.1.1` por `127.0.1.1   infra.g07rc2.com.br infra`.
   Verificação: `hostname` (`infra`) e `hostname -f` (`infra.g07rc2.com.br`).

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
   - A partir daqui o e-mail e o cliente conseguem usar o `apt-get`.

4. **DHCP**
   ```bash
   apt-get install isc-dhcp-server
   cp infra/isc-dhcp-server /etc/default/isc-dhcp-server
   mv /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.original
   cp infra/dhcpd.conf /etc/dhcp/dhcpd.conf
   service isc-dhcp-server restart
   systemctl enable isc-dhcp-server
   ```
   Ajuste o MAC da reserva em `dhcpd.conf` para o MAC real do cliente (`ip a` no cliente, linha `link/ether`).

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
   Em `/etc/resolv.conf` da infra: `search g07rc2.com.br` e `nameserver 127.0.0.1`. Depois de reiniciar, confira o arquivo: se o `dhclient` da `enp0s3` sobrescrever, reponha as duas linhas.
   Registros: `A` de `infra`, `www`, `intranet` e `mail`; `MX` para `mail.g07rc2.com.br`; `PTR` de `1.0` (infra) e `2.0` (mail) na zona reversa.

6. **SSH com chave**
   ```bash
   apt-get install ssh
   cp infra/sshd_config /etc/ssh/sshd_config
   sshd -t && service ssh restart
   ```
   Se o `sshd_config` desativa a senha, copie antes a chave do cliente (seção 5.2) ou mantenha o console da VM aberto. Senão você perde o acesso.

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

8. **CA local e certificados**
   ```bash
   apt-get install openssl ssl-cert
   ```
   Gerar conforme a seção 5.1 e instalar `web.crt`/`web.key` em `/etc/ssl/certs/` e `/etc/ssl/private/`.
   Deixe a CA **pública** à disposição do cliente:
   ```bash
   cp ~/ca/ca.crt /home/aluno/ca.crt
   chown aluno:aluno /home/aluno/ca.crt
   ```
   O `mail.crt` e o `mail.key` serão enviados ao e-mail no item 4.2.3.

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
   Cada `ServerName` precisa existir no DNS e constar no `subjectAltName` do certificado.
   Os conteúdos dos sites estão em `infra/www/` e `infra/intranet/` (`<PREENCHER, se incluir no repositório>`).

### 4.2 Servidor de e-mail

Pré-requisitos na infra: NAT (4.1.3), DNS com `mail` e MX (4.1.5) e CA com `mail.crt` e `mail.key` (4.1.8).

1. **Nome e rede:** `hostnamectl set-hostname mail`. Em `/etc/hosts`, mantenha `127.0.0.1 localhost` e substitua a linha `127.0.1.1` por `127.0.1.1   mail.g07rc2.com.br mail`. Copie `mail/interfaces` e ajuste `/etc/resolv.conf` (`search g07rc2.com.br`, `nameserver 172.16.0.1`). Reinicie.
   Verificação: `hostname` (`mail`), `hostname -f` (`mail.g07rc2.com.br`), `ping 172.16.0.1` e `ping 8.8.8.8`.
2. **Pacotes e usuário de acesso**
   ```bash
   apt-get update
   apt-get install postfix dovecot-imapd rsyslog ssh
   adduser aluno
   ```
   No Postfix: "Internet Site", `g07rc2.com.br`. O `ssh` é instalado agora porque a infra envia o certificado por `scp` no próximo passo, e o usuário `aluno` recebe o arquivo e a chave do cliente (o root não entra por SSH).
3. **Certificado do e-mail:** gerado na CA da infra (seção 5.1). Na **infra**:
   ```bash
   scp ~/ca/mail.crt ~/ca/mail.key aluno@172.16.0.2:/tmp/
   ```
   No **e-mail**, como root:
   ```bash
   mv /tmp/mail.crt /etc/ssl/certs/mail.crt
   mv /tmp/mail.key /etc/ssl/private/mail.key
   chmod 640 /etc/ssl/private/mail.key
   chgrp ssl-cert /etc/ssl/private/mail.key
   ```
4. **Configuração:** copie `mail/main.cf` e `mail/master.cf` para `/etc/postfix/`, e os arquivos de `mail/dovecot/` para `/etc/dovecot/`. O `myhostname` do Postfix deve ser `mail.g07rc2.com.br`.
5. **Contas e serviços**
   ```bash
   adduser ana
   adduser bia
   service postfix restart
   service dovecot restart
   systemctl enable postfix dovecot
   ```
6. **SSH:** o `ssh` já foi instalado no item 2. Falta copiar a chave do cliente (seção 5.2) e, só depois de testar a entrada sem senha, desativar a senha aqui, como em 4.1.6.

### 4.3 Estação cliente

1. Interface cabeada em **DHCP** (sem IP manual). O cliente mantém o nome que já tem: não é preciso renomear nem registrar no DNS.
2. Instale os pacotes da seção 2.
3. Instale a CA. Use o `ca/ca.crt` do repositório ou busque na infra:
   ```bash
   scp aluno@172.16.0.1:ca.crt ~/       # opcional, se não usar o do repositório
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
ssh-copy-id aluno@172.16.0.1
ssh-copy-id aluno@172.16.0.2     # só depois do item 4.2.2 (cria o usuário aluno no e-mail)
```
O usuário `aluno` precisa existir nos dois servidores.

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
2. `dig www.g07rc2.com.br`, `dig intranet.g07rc2.com.br`, `dig g07rc2.com.br MX`, `dig -x 172.16.0.1`, `dig -x 172.16.0.2`, `dig naoexiste.g07rc2.com.br` (NXDOMAIN).
3. `ssh aluno@172.16.0.1` (por chave) e `smbclient //172.16.0.1/diretorio_grupos -U aluno` (permitido) / `-U professor` (negado).
4. Navegador: `http://www.g07rc2.com.br` redireciona para HTTPS; `https://intranet.g07rc2.com.br` pede autenticação; certificado válido.
5. Thunderbird: enviar de `ana@g07rc2.com.br` para `bia@g07rc2.com.br` e ler por IMAP; logs em `/var/log/mail.log` no servidor de e-mail.

**Persistência:** reinicie as três máquinas e repita os testes. Ligue nesta ordem: infra (NAT, DHCP, DNS), depois o e-mail e por último o cliente. Conferir também `systemctl is-enabled ...` e `cat /etc/iptables/rules.v4`.

---

## 7. Capturas de pacotes

Em [`capturas/`](capturas/): `<PREENCHER: nomes dos arquivos .pcap>` e os comentários explicando as etapas do protocolo (DHCP: Discover/Offer/Request/Ack; DNS: query/response; TLS: handshake).

## 8. Relatório técnico

[`relatorio.pdf`](relatorio.pdf): decisões, problemas encontrados e correções.