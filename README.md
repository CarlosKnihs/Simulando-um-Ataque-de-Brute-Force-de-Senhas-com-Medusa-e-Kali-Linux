# Simulando um Ataque de Brute Force de Senhas com Medusa e Kali Linux

Projeto prático desenvolvido para o desafio de código da DIO, com o objetivo de simular ataques de força bruta em ambiente controlado, utilizando Kali Linux, a ferramenta Medusa e máquinas vulneráveis (Metasploitable 2 e DVWA).

## Descrição do Desafio

Implementar, documentar e compartilhar um projeto prático utilizando Kali Linux e a ferramenta Medusa, em conjunto com ambientes vulneráveis (por exemplo, Metasploitable 2 e DVWA), para simular cenários de ataque de força bruta e exercitar medidas de prevenção.

- Configurar o ambiente: duas VMs (Kali Linux e Metasploitable 2) no VirtualBox, com rede interna (host-only).
- Executar ataques simulados: força bruta em FTP, automação de tentativas em formulário web (DVWA) e password spraying em SMB com enumeração de usuários.
- Documentar os testes: wordlists simples, comandos utilizados, validação de acessos e recomendações de mitigação.

---

## Montagem do Desafio

### Ambiente de Testes

| Máquina         | IP                 | Função                          |
| ---------------- | ------------------ | --------------------------------- |
| Kali Linux       | `192.168.56.102` | Atacante                          |
| Metasploitable 2 | `192.168.56.101` | Alvo vulnerável (FTP, SMB, DVWA) |

- Hypervisor: VirtualBox
- Rede: Host-only (isolada da internet e da rede física)
- DVWA: hospedado na própria Metasploitable 2, acessível em `http://192.168.56.101/dvwa`, com nível de segurança **low**

> Todos os testes foram realizados em laboratório próprio, isolado, sem exposição a redes externas ou a terceiros.

### Estrutura do Repositório

```
.
├── README.md
├── wordlists/
│   ├── usuarios.txt
│   ├── senhas.txt
│   └── smb_usuarios.txt
└── images/
    ├── ping.png
    ├── nmap-scan.png
    ├── ftp-test.png
    ├── ftp-brute.png
    ├── ftp-successful.png
    ├── dvwa-login.png
    ├── dvwa-cookie.png
    ├── dvwa-brute.png
    ├── dvwa-success.png
    ├── smb-users.png
    ├── smb-brute.png
    └── smb-success.png
```

### Etapa 1 – Verificação de Conectividade

Inicialmente vamos identificar se a máquina está acessível usando o comando ping.

Evidência: `images/ping.png`

### Etapa 2 – Reconhecimento com Nmap

Antes de qualquer ataque, mapeie os serviços ativos no alvo:

```bash
nmap -sV -p- 192.168.56.101
```

Serviços relevantes esperados na Metasploitable 2: `21/ftp`, `139,445/smb`, `80/http` (DVWA).

Evidência: `images/nmap-scan.png`

### Etapa 3 – Força Bruta em FTP

Visualizando o resultado da etapa anterior, é possível identificar que a porta 21 (FTP) está aberta e ainda obtivemos a sua **Versão 2.3.4**. Com isso, vamos iniciar um ataque de força bruta neste serviço.

Vamos verificar se o serviço está ativo e aceitando conexões com o seguinte comando:

```bash
ftp 192.168.56.101
```

Evidência: `images/ftp-test.png`

Identificamos que o serviço está funcionando. Para iniciar o ataque de força bruta, necessitamos criar dois arquivos, um com uma lista de usuários, pois, não sabemos qual usuário está cadastrado e uma outra lista de senhas.

Para criar a lista de usuários usaremos o comando abaixo:

```bash
echo -e "msfadmin\nadmin\nuser\nroot\nservice" > usuarios.txt
```

Agora precisamos criar a lista de senhas:

```bash
echo -e "123456\npassword\nmsfadmin\nadmin\ntoor\nletmein\nsenha123" > senhas.txt
```

Agora com estes dois arquivos, podemos iniciar o ataque usando o comando:

```bash
medusa -h 192.168.56.101 -U usuarios.txt -P senhas.txt -M ftp -t 4 -f
```

Parâmetros:

- `-h` IP do alvo
- `-U` / `-P` wordlists de usuários e senhas
- `-M ftp` módulo de FTP
- `-t 4` número de threads (tentativas em paralelo)
- `-f` interrompe ao encontrar a primeira credencial válida

Ao terminar é possível identificar a senha que houve sucesso

Evidência: `images/ftp-brute.png`

**Resultado:** `msfadmin:msfadmin`

Vamos testar o resultado obtido.

Evidência: `images/ftp-successful.png`

### Etapa 4 – Força Bruta no Formulário Web (DVWA)

Acesse `http://192.168.56.101/dvwa/vulnerabilities/brute/`, abra as ferramentas de desenvolvedor do navegador (F12), coloque na aba "Network", e faça login com uma credencial qualquer somente para obter os dados necessários para a requisição.

Evidência: `images/dvwa-login.png`

Capture o cookie de sessão (`PHPSESSID`) pelas ferramentas de desenvolvedor do navegador.

Evidência: `images/dvwa-cookie.png`

Execute o comando medusa:

```bash
medusa -h 192.168.56.101 -U usuarios.txt -P senhas.txt -M web-form \
  -m FORM:"/dvwa/vulnerabilities/brute/" \
  -m FORM-DATA:"GET?username=&password=&Login=Login" \
  -m DENY-SIGNAL:"Username and/or password incorrect." \
  -m CUSTOM-HEADER:"Cookie: PHPSESSID=64cd6bfb344a15844238dbc2269f6c76; security=low" \
  -t 4 -f
```

Evidência: `images/dvwa-brute.png`

Após a execução é possível visualizar que houve uma tentativa com sucesso.

**Resultado:** `User: admin` / `Password: password`

Vamos testar o acesso com o resultado obtido.

Evidência: `images/dvwa-success.png`

### Etapa 5 – Password Spraying em SMB

Voltando a visualizar o resultado do nmap da etapa 1, é possível identificar que as portas 139 e 445 também estão abertas, então vamos testá-las.

Iniciaremos utilizando a ferramenta *enum4linux*, que efetua técnicas de enumeração; esse resultado vamos salvar em um arquivo para visualização posterior.

```bash
enum4linux -a 192.168.56.101 | tee enum_output.txt
```

Após o término da execução, vamos ler o arquivo; neste momento vamos focar somente nos usuários.

Evidência: `images/smb-users.png`

Vamos criar uma lista dos usuários mais comuns:

```bash
echo -e "msfadmin\nuser\nroot\nservice" > smb_usuarios.txt
```

A lista de senhas, vamos utilizar a que já temos por já conter senhas padrões. Agora vamos usar novamente a ferramenta medusa para testar os usuários e as senhas

```bash
medusa -h 192.168.56.101 -U smb_usuarios.txt -P senhas.txt -M smbnt -t 4 -f
```

Evidência: `images/smb-brute.png`

**Resultado:** `User: msfadmin` / `Password: msfadmin`

Vamos testar o resultado:

```bash
smbclient -L //192.168.56.101 -U msfadmin
```

Evidência: `images/smb-success.png`

## Wordlists Utilizadas

- [`usuarios.txt`](wordlists/usuarios.txt): usuários comuns e conhecidos da Metasploitable 2.
- [`senhas.txt`](wordlists/senhas.txt): senhas fracas/padrão comumente usadas em testes de força bruta.
- [`smb_usuarios.txt`](wordlists/usuarios.txt): usuários comuns localizados na enumeração da ferramenta enum4linux.

> Wordlists reduzidas e propositalmente simples, com foco didático. Em uma auditoria real, utilize listas maiores (ex.: `rockyou.txt`) e sempre dentro do escopo autorizado.

## Medidas de Mitigação Recomendadas

- **Políticas de senha forte**: exigir senhas com tamanho mínimo, complexidade e sem reutilização de valores padrão (ex.: `msfadmin`, `admin`, `123456`).
- **Bloqueio por tentativas (account lockout)**: bloquear temporariamente a conta após um número definido de tentativas falhas.
- **Rate limiting / CAPTCHA**: limitar a taxa de requisições de login em formulários web e adicionar CAPTCHA após falhas repetidas.
- **Autenticação multifator (MFA)**: reduzir o impacto de uma senha comprometida.
- **Desativar serviços não utilizados**: FTP e SMB sem necessidade de uso devem ser desativados ou restritos por firewall/VPN.
- **Monitoramento e alertas**: logs de autenticação e alertas (ex.: Fail2ban, SIEM) para identificar tentativas de força bruta em tempo real.
- **Segmentação de rede**: isolar serviços críticos e limitar o acesso apenas a hosts autorizados.

## Considerações Finais

Este projeto teve finalidade exclusivamente educacional, executado em ambiente virtualizado e isolado (rede host-only), sem impacto em sistemas de terceiros. O objetivo foi compreender, na prática, como ataques de força bruta funcionam e quais controles reduzem esse risco em ambientes reais.
