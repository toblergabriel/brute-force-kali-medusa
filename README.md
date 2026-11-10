# Ataques de Força Bruta com Kali Linux e Medusa (Ambiente Controlado)

Projeto prático desenvolvido como parte do desafio da DIO sobre auditoria de segurança com **Kali Linux** e **Medusa**, simulando ataques de força bruta em serviços **FTP**, **DVWA (Web)** e **SMB**, em um ambiente totalmente controlado e isolado.

> ⚠️ **Aviso:** todos os testes foram realizados em máquinas virtuais de laboratório (Metasploitable 2, DVWA), isoladas em rede host-only, sem acesso à internet. Técnicas de força bruta nunca devem ser aplicadas contra sistemas de terceiros sem autorização explícita — isso é crime.

## Objetivo

Compreender na prática como funcionam ataques de força bruta/dicionário contra diferentes serviços (FTP, formulário web, SMB), usar o Medusa para automatizar essas tentativas, e documentar tanto os resultados quanto os erros cometidos no processo — que no fim foram tão instrutivos quanto os ataques em si.

## Ambiente

| Item | Detalhe |
|---|---|
| Atacante | Kali Linux (VM, VirtualBox) |
| Alvo | Metasploitable 2 (VM, VirtualBox) |
| Rede | Host-only (isolada) |
| IP do alvo | `192.168.56.102` |
| Ferramentas | Medusa, enum4linux |

## Wordlists utilizadas

- [`users.txt`](./wordlists/users.txt) — usuários testados contra DVWA: `user`, `msfadmin`, `admin`, `root`
- [`pass.txt`](./wordlists/pass.txt) — senhas testadas contra DVWA: `123456`, `password`, `qwert`, `msfadmin`
- [`smb_users.txt`](./wordlists/smb_users.txt) — usuários para password spray em SMB: `user`, `msfadmin`, `service`
- [`senhas_spray.txt`](./wordlists/senhas_spray.txt) — senhas para o spray em SMB

> **Nota de aprendizado:** a wordlist `senhas_spray.txt` originalmente tinha um erro de digitação — duas senhas (`123456` e `welcome123`) ficaram coladas em uma linha só, separadas por `\`, e uma entrada estava truncada (`passwor` em vez de `password`). Já corrigido na versão atual do arquivo. É um bom exemplo de como um erro bobo de digitação/quebra de linha pode invalidar silenciosamente um teste inteiro: o Medusa testaria uma senha que não existe em lugar nenhum e nunca encontraria a real.

## Cenário 1 — Força bruta em FTP

```bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M ftp
```

![Força bruta FTP](./images/ftp-bruteforce.png)

Na primeira tentativa, o IP foi digitado errado (`192.168.0.102` em vez de `192.168.56.102`), resultando em `port 21 was not open`. Após corrigir o IP, o ataque rodou normalmente.

**Resultado:** credencial válida encontrada — `msfadmin:msfadmin` `[SUCCESS]`.

## Cenário 2 — Automação de tentativas em formulário Web (DVWA)

Ataque de dicionário contra o formulário de login do DVWA (`/dvwa/login.php`), usando o módulo HTTP do Medusa.

```bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M http \
  -m PAGE:'/dvwa/login.php' \
  -m FORM:'username=USER&password=PASS&Login=Login' \
  -m FAIL='Login failed' -t 6
```

### Erro encontrado e lição aprendida

Na primeira tentativa, o comando foi colado no terminal quebrado em várias linhas com `\` no final. Por causa de um espaço extra depois da barra (ou comportamento de paste do terminal), o shell não interpretou a continuação de linha corretamente. O resultado foi que `medusa` rodou **sem os parâmetros `-m PAGE`, `-m FORM` e `-m FAIL`**, e o shell tentou executar `-m` como comando (gerando `-m: comando não encontrado` repetidas vezes).

Sem o parâmetro `FAIL`, o Medusa não tinha como saber diferenciar um login bem-sucedido de um malsucedido — e por isso **todas as 4 contas testadas (`user`, `msfadmin`, `admin`, `root`) foram marcadas como `SUCCESS`**, o que era um falso positivo.

**Correção:** reescrever o comando em uma única linha (evita problemas de paste/continuação) e confirmar que os três parâmetros do módulo HTTP (`PAGE`, `FORM`, `FAIL`) estão sendo passados corretamente.

### Segundo erro: sintaxe do FORM

Mesmo escrevendo numa linha só, uma segunda tentativa ainda falhou:

```
-m FORM: 'username="USER"&password="PASS"Login=Login'
```

![Erro de sintaxe no FORM do DVWA](./images/dvwa-medusa-erro2.png)

Dois problemas na string do `FORM`: havia aspas literais sobrando em volta dos placeholders `USER` e `PASS` (não deveriam estar lá), e faltava um `&` separando `password=PASS` de `Login=Login`. O Medusa não conseguiu interpretar os parâmetros (`WARNING: Invalid method: PAGE/FORM/FAIL.failed`), rodou o módulo HTTP sem validação real de sucesso/falha, e **todas as combinações de usuário/senha testadas voltaram como `[SUCCESS]`** — um segundo falso positivo, por um motivo diferente do primeiro erro.

**Sintaxe corrigida** (sem aspas nos placeholders, `&` separando todos os campos do formulário):

```bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M http -m PAGE:'/dvwa/login.php' -m FORM:'username=USER&password=PASS&Login=Login' -m FAIL='Login failed' -t 6
```

### Terceiro erro: espaço entre a flag e o valor

Mesmo corrigindo o `FORM`, uma nova tentativa ainda deu o mesmo sintoma:

![Terceiro erro de sintaxe no DVWA](./images/dvwa-medusa-erro2.png)

Dessa vez a causa era mais sutil: havia um **espaço entre os dois-pontos e as aspas** (`PAGE: '/dvwa/login.php'` em vez de `PAGE:'/dvwa/login.php'`). Como o shell separa argumentos por espaço, o `-m` recebeu só `PAGE:` como valor, e o restante virou um argumento solto — por isso o Medusa voltou a acusar `WARNING: Invalid method: PAGE/FORM/FAIL.failed` e, de novo, **as 16 combinações testadas (4 usuários × 4 senhas) voltaram como `[SUCCESS]`**.

> **Nota sobre este resultado:** este print documenta o terceiro erro de sintaxe encontrado ao tentar configurar o módulo HTTP do Medusa — não um resultado de teste válido. As 16 contas marcadas como `SUCCESS` são um falso positivo decorrente do módulo rodando sem os parâmetros `PAGE`/`FORM`/`FAIL` corretamente interpretados, e não indicam que o DVWA aceita qualquer credencial. Este cenário ficou registrado como parte do processo de aprendizado (três erros de sintaxe distintos gerando o mesmo sintoma), sem um ataque HTTP bem-sucedido confirmado neste laboratório.

## Cenário 3 — Enumeração e Password Spraying em SMB

### Enumeração de usuários com `enum4linux`

```bash
enum4linux 192.168.56.102
```

A enumeração anônima (sessão NULL) foi aceita pelo servidor Samba (`Samba 3.0.20-Debian`), permitindo extrair:

- **Lista completa de usuários locais** via RPC, incluindo `root`, `msfadmin`, `service`, `user`, `www-data`, `postgres`, `mysql`, entre outros (34 contas no total)
- **Política de senha:** sem bloqueio de conta por tentativas (`Account Lockout Threshold: None`), sem complexidade exigida, comprimento mínimo de apenas 5 caracteres
- **Compartilhamento `tmp` listável anonimamente** (`Mapping: OK / Listing: OK`)

Essa enumeração é o que tornou possível montar a wordlist [`smb_users.txt`](./wordlists/smb_users.txt) com usuários reais do sistema, em vez de adivinhar nomes.

### Password Spraying

```bash
medusa -h 192.168.56.102 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 2 -T 50
```

![Password spray SMB](./images/smb-spray.png)

**Resultado final:**

| Usuário | Senhas testadas | Resultado |
|---|---|---|
| `user` | password, 123456, welcome123, msfadmin | nenhuma funcionou |
| `msfadmin` | password, 123456, welcome123, msfadmin | **`msfadmin:msfadmin` `[SUCCESS (ADMIN$ – Access Allowed)]`** |
| `service` | password, 123456, welcome123, msfadmin | nenhuma funcionou |

A credencial `msfadmin:msfadmin` foi validada com sucesso no SMB, consistente com o mesmo usuário/senha já encontrado no cenário de FTP — reforçando que essa conta reaproveita a senha padrão em múltiplos serviços, e que o acesso obtido via SMB já é suficiente para administração (`ADMIN$ – Access Allowed`).

## Recomendações de Mitigação

Com base nas vulnerabilidades exploradas neste laboratório:

| Vulnerabilidade observada | Mitigação recomendada |
|---|---|
| Sem bloqueio de conta após tentativas falhas (SMB) | Configurar `Account Lockout Threshold` (ex: 5 tentativas) |
| Senha mínima de 5 caracteres, sem exigência de complexidade | Política de senha forte (mínimo 12+ caracteres, complexidade obrigatória) |
| Sessão NULL/anônima permitida no SMB | Desabilitar sessões anônimas (`restrict anonymous` no Samba) |
| Login HTTP sem limite de tentativas (DVWA) | Rate limiting, CAPTCHA após N tentativas, bloqueio temporário de IP |
| Senhas fracas e previsíveis nas wordlists testadas | Treinamento de usuários, gerenciador de senhas, MFA/2FA |
| Compartilhamento SMB listável sem autenticação | Restringir permissões de share, exigir autenticação |

## Aprendizados

- Ferramentas de automação como o Medusa são tão boas quanto os parâmetros que você passa para elas — um erro de sintaxe no shell pode gerar resultados falsos positivos silenciosos, por isso validar manualmente ao menos uma credencial "encontrada" é essencial antes de confiar no resultado.
- Enumeração (reconhecimento) antes do ataque de força bruta reduz drasticamente o espaço de busca: em vez de adivinhar usuários, o `enum4linux` revelou contas reais via sessão NULL do SMB.
- Políticas de senha fracas e ausência de lockout são, na prática, o que torna um ataque de força bruta viável — a ferramenta é só o motor, a vulnerabilidade real está na configuração do alvo.
- A sintaxe do módulo HTTP do Medusa é sensível a detalhes pequenos (aspas a mais, `&` faltando entre campos, até um espaço solto entre a flag e o valor). Três erros de digitação diferentes no mesmo comando geraram o mesmo sintoma — todas as credenciais testadas aparecendo como `SUCCESS` — o que reforça a importância de **sempre validar manualmente** pelo menos uma credencial "encontrada" antes de confiar no resultado de uma ferramenta automatizada, e de saber reconhecer um falso positivo em vez de reportá-lo como achado real.
- Credenciais padrão reaproveitadas entre serviços (`msfadmin:msfadmin` válida tanto em FTP quanto em SMB) são um risco real em ambientes corporativos: comprometer um serviço pode abrir caminho para outros.

---

*Projeto desenvolvido para fins educacionais, como parte do desafio "Explorando Vulnerabilidades com Kali Linux e Medusa" da [DIO](https://www.dio.me/).*
