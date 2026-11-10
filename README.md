# Ataques de Força Bruta com Kali Linux e Medusa (Ambiente Controlado)
# Brute-Force Attacks with Kali Linux and Medusa (Controlled Environment)

🇧🇷 Projeto prático desenvolvido como parte do desafio da DIO sobre auditoria de segurança com **Kali Linux** e **Medusa**, simulando ataques de força bruta em serviços **FTP**, **DVWA (Web)** e **SMB**, em um ambiente totalmente controlado e isolado.

🇺🇸 Hands-on project built for a DIO security-auditing challenge using **Kali Linux** and **Medusa**, simulating brute-force attacks against **FTP**, **DVWA (Web)**, and **SMB** services in a fully controlled, isolated lab environment.

> ⚠️ **Aviso / Warning:** todos os testes foram realizados em máquinas virtuais de laboratório (Metasploitable 2, DVWA), isoladas em rede host-only, sem acesso à internet. Técnicas de força bruta nunca devem ser aplicadas contra sistemas de terceiros sem autorização explícita — isso é crime. | All tests were performed on isolated lab VMs (Metasploitable 2, DVWA) on a host-only network with no internet access. Brute-force techniques must never be used against third-party systems without explicit authorization — doing so is a crime.

## Objetivo / Objective

🇧🇷 Compreender na prática como funcionam ataques de força bruta/dicionário contra diferentes serviços (FTP, formulário web, SMB), usar o Medusa para automatizar essas tentativas, e documentar tanto os resultados quanto os erros cometidos no processo — que no fim foram tão instrutivos quanto os ataques em si.

🇺🇸 Understand hands-on how brute-force/dictionary attacks work against different services (FTP, web form, SMB), use Medusa to automate the attempts, and document both the results and the mistakes made along the way — which turned out to be just as instructive as the attacks themselves.

## Ambiente / Environment

| Item | Detalhe / Detail |
|---|---|
| Atacante / Attacker | Kali Linux (VM, VirtualBox) |
| Alvo / Target | Metasploitable 2 (VM, VirtualBox) |
| Rede / Network | Host-only (isolada / isolated) |
| IP do alvo / Target IP | `192.168.56.102` |
| Ferramentas / Tools | Medusa, enum4linux |

## Wordlists utilizadas / Wordlists used

- [`users.txt`](./wordlists/users.txt) — usuários testados contra o DVWA / users tested against DVWA: `user`, `msfadmin`, `admin`, `root`
- [`pass.txt`](./wordlists/pass.txt) — senhas testadas contra o DVWA / passwords tested against DVWA: `123456`, `password`, `qwert`, `msfadmin`
- [`smb_users.txt`](./wordlists/smb_users.txt) — usuários para o password spray em SMB / users for the SMB password spray: `user`, `msfadmin`, `service`
- [`senhas_spray.txt`](./wordlists/senhas_spray.txt) — senhas para o spray em SMB / passwords for the SMB spray

> **Nota de aprendizado / Lesson learned:** 🇧🇷 a wordlist `senhas_spray.txt` originalmente tinha um erro de digitação — duas senhas (`123456` e `welcome123`) ficaram coladas em uma linha só, separadas por `\`, e uma entrada estava truncada (`passwor` em vez de `password`). Já corrigido na versão atual do arquivo. É um bom exemplo de como um erro bobo de digitação/quebra de linha pode invalidar silenciosamente um teste inteiro. | 🇺🇸 the `senhas_spray.txt` wordlist originally had a typo — two passwords (`123456` and `welcome123`) got merged into a single line, separated by `\`, and one entry was truncated (`passwor` instead of `password`). Already fixed in the current version of the file. A good example of how a small typo/line-break mistake can silently invalidate an entire test.

## Cenário 1 — Força bruta em FTP / Scenario 1 — FTP Brute Force

```bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M ftp
```

![Força bruta FTP / FTP brute force](./images/ftp-bruteforce.png)

🇧🇷 Na primeira tentativa, o IP foi digitado errado (`192.168.0.102` em vez de `192.168.56.102`), resultando em `port 21 was not open`. Após corrigir o IP, o ataque rodou normalmente.

🇺🇸 On the first attempt, the IP was mistyped (`192.168.0.102` instead of `192.168.56.102`), resulting in `port 21 was not open`. After fixing the IP, the attack ran normally.

**Resultado / Result:** credencial válida encontrada / valid credential found — `msfadmin:msfadmin` `[SUCCESS]`.

## Cenário 2 — Automação de tentativas em formulário Web (DVWA) / Scenario 2 — Web Form Attack Automation (DVWA)

🇧🇷 Ataque de dicionário contra o formulário de login do DVWA (`/dvwa/login.php`), usando o módulo HTTP do Medusa. | 🇺🇸 Dictionary attack against the DVWA login form (`/dvwa/login.php`), using Medusa's HTTP module.

```bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M http \
  -m PAGE:'/dvwa/login.php' \
  -m FORM:'username=USER&password=PASS&Login=Login' \
  -m FAIL='Login failed' -t 6
```

### Erro encontrado e lição aprendida / Error found and lesson learned

🇧🇷 Na primeira tentativa, o comando foi colado no terminal quebrado em várias linhas com `\` no final. Por causa de um espaço extra depois da barra (ou comportamento de paste do terminal), o shell não interpretou a continuação de linha corretamente. O resultado foi que `medusa` rodou **sem os parâmetros `-m PAGE`, `-m FORM` e `-m FAIL`**, e o shell tentou executar `-m` como comando (gerando `-m: comando não encontrado` repetidas vezes).

🇺🇸 On the first attempt, the command was pasted into the terminal split across several lines ending in `\`. Because of an extra space after the backslash (or how the terminal handled the paste), the shell didn't interpret the line continuation correctly. As a result, `medusa` ran **without the `-m PAGE`, `-m FORM`, and `-m FAIL` parameters**, and the shell tried to run `-m` as a command (producing repeated `-m: command not found` errors).

🇧🇷 Sem o parâmetro `FAIL`, o Medusa não tinha como saber diferenciar um login bem-sucedido de um malsucedido — e por isso **todas as 4 contas testadas (`user`, `msfadmin`, `admin`, `root`) foram marcadas como `SUCCESS`**, o que era um falso positivo.

🇺🇸 Without the `FAIL` parameter, Medusa had no way to tell a successful login from a failed one — so **all 4 tested accounts (`user`, `msfadmin`, `admin`, `root`) were flagged as `SUCCESS`**, which was a false positive.

**Correção / Fix:** reescrever o comando em uma única linha (evita problemas de paste/continuação) e confirmar que os três parâmetros do módulo HTTP (`PAGE`, `FORM`, `FAIL`) estão sendo passados corretamente. | rewrite the command on a single line (avoids paste/continuation issues) and confirm the three HTTP module parameters (`PAGE`, `FORM`, `FAIL`) are being passed correctly.

### Segundo erro: sintaxe do FORM / Second error: FORM syntax

🇧🇷 Mesmo escrevendo numa linha só, uma segunda tentativa ainda falhou: | 🇺🇸 Even written as a single line, a second attempt still failed:


![Erro de sintaxe no FORM do DVWA / DVWA FORM syntax error](./images/dvwa-medusa-erro2.png)

🇧🇷 Dois problemas na string do `FORM`: havia aspas literais sobrando em volta dos placeholders `USER` e `PASS` (não deveriam estar lá), e faltava um `&` separando `password=PASS` de `Login=Login`. O Medusa não conseguiu interpretar os parâmetros (`WARNING: Invalid method: PAGE/FORM/FAIL.failed`), rodou o módulo HTTP sem validação real de sucesso/falha, e **todas as combinações de usuário/senha testadas voltaram como `[SUCCESS]`** — um segundo falso positivo, por um motivo diferente do primeiro erro.

🇺🇸 Two problems in the `FORM` string: there were stray literal quotes around the `USER` and `PASS` placeholders (they shouldn't be there), and an `&` was missing between `password=PASS` and `Login=Login`. Medusa couldn't parse the parameters (`WARNING: Invalid method: PAGE/FORM/FAIL.failed`), ran the HTTP module with no real success/failure validation, and **every username/password combination tested came back as `[SUCCESS]`** — a second false positive, caused by a different mistake than the first one.

**Sintaxe corrigida / Corrected syntax** (sem aspas nos placeholders, `&` separando todos os campos / no quotes around placeholders, `&` separating every field):

```bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M http -m PAGE:'/dvwa/login.php' -m FORM:'username=USER&password=PASS&Login=Login' -m FAIL='Login failed' -t 6
```

### Terceiro erro: espaço entre a flag e o valor / Third error: space between flag and value

🇧🇷 Mesmo corrigindo o `FORM`, uma nova tentativa ainda deu o mesmo sintoma: | 🇺🇸 Even after fixing the `FORM`, a new attempt still showed the same symptom:

![Terceiro erro de sintaxe no DVWA / Third DVWA syntax error](./images/dvwa-medusa-erro2.png)

🇧🇷 Dessa vez a causa era mais sutil: havia um **espaço entre os dois-pontos e as aspas** (`PAGE: '/dvwa/login.php'` em vez de `PAGE:'/dvwa/login.php'`). Como o shell separa argumentos por espaço, o `-m` recebeu só `PAGE:` como valor, e o restante virou um argumento solto — por isso o Medusa voltou a acusar os mesmos warnings e, de novo, **as 16 combinações testadas (4 usuários × 4 senhas) voltaram como `[SUCCESS]`**.

🇺🇸 This time the cause was subtler: there was a **space between the colon and the quote** (`PAGE: '/dvwa/login.php'` instead of `PAGE:'/dvwa/login.php'`). Since the shell splits arguments by whitespace, `-m` only received `PAGE:` as its value, and the rest became a stray argument — so Medusa raised the same warnings again, and once more **all 16 combinations tested (4 users × 4 passwords) came back as `[SUCCESS]`**.

> **Nota sobre este resultado / Note on this result:** 🇧🇷 este print documenta o terceiro erro de sintaxe encontrado ao tentar configurar o módulo HTTP do Medusa — não um resultado de teste válido. As 16 contas marcadas como `SUCCESS` são um falso positivo, e não indicam que o DVWA aceita qualquer credencial. Este cenário ficou registrado como parte do processo de aprendizado, sem um ataque HTTP bem-sucedido confirmado neste laboratório. | 🇺🇸 this screenshot documents the third syntax error found while trying to configure Medusa's HTTP module — not a valid test result. The 16 accounts flagged as `SUCCESS` are a false positive and do not mean DVWA accepts any credential. This scenario was kept as part of the learning process, with no confirmed successful HTTP attack in this lab.

## Cenário 3 — Enumeração e Password Spraying em SMB / Scenario 3 — SMB Enumeration and Password Spraying

### Enumeração de usuários com `enum4linux` / User enumeration with `enum4linux`

```bash
enum4linux 192.168.56.102
```

🇧🇷 A enumeração anônima (sessão NULL) foi aceita pelo servidor Samba (`Samba 3.0.20-Debian`), permitindo extrair: | 🇺🇸 Anonymous enumeration (NULL session) was accepted by the Samba server (`Samba 3.0.20-Debian`), allowing the extraction of:

- **Lista completa de usuários locais / Full list of local users** via RPC, incluindo / including `root`, `msfadmin`, `service`, `user`, `www-data`, `postgres`, `mysql`, entre outros / among others (34 contas no total / 34 accounts total)
- **Política de senha / Password policy:** sem bloqueio de conta por tentativas / no account lockout on failed attempts (`Account Lockout Threshold: None`), sem complexidade exigida / no required complexity, comprimento mínimo de apenas 5 caracteres / minimum length of only 5 characters
- **Compartilhamento `tmp` listável anonimamente / `tmp` share listable anonymously** (`Mapping: OK / Listing: OK`)

🇧🇷 Essa enumeração é o que tornou possível montar a wordlist [`smb_users.txt`](./wordlists/smb_users.txt) com usuários reais do sistema, em vez de adivinhar nomes. | 🇺🇸 This enumeration is what made it possible to build the [`smb_users.txt`](./wordlists/smb_users.txt) wordlist with real system users, instead of guessing names.

### Password Spraying

```bash
medusa -h 192.168.56.102 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 2 -T 50
```

![Password spray SMB](./images/smb-spray.png)

**Resultado final / Final result:**

| Usuário / User | Senhas testadas / Passwords tested | Resultado / Result |
|---|---|---|
| `user` | password, 123456, welcome123, msfadmin | nenhuma funcionou / none worked |
| `msfadmin` | password, 123456, welcome123, msfadmin | **`msfadmin:msfadmin` `[SUCCESS (ADMIN$ – Access Allowed)]`** |
| `service` | password, 123456, welcome123, msfadmin | nenhuma funcionou / none worked |

🇧🇷 A credencial `msfadmin:msfadmin` foi validada com sucesso no SMB, consistente com o mesmo usuário/senha já encontrado no cenário de FTP — reforçando que essa conta reaproveita a senha padrão em múltiplos serviços, e que o acesso obtido via SMB já é suficiente para administração (`ADMIN$ – Access Allowed`).

🇺🇸 The `msfadmin:msfadmin` credential was successfully validated over SMB, consistent with the same username/password already found in the FTP scenario — reinforcing that this account reuses the same default password across multiple services, and that the access gained via SMB is already enough for administration (`ADMIN$ – Access Allowed`).

## Recomendações de Mitigação / Mitigation Recommendations

| Vulnerabilidade observada / Observed vulnerability | Mitigação recomendada / Recommended mitigation |
|---|---|
| Sem bloqueio de conta após tentativas falhas (SMB) / No account lockout after failed attempts (SMB) | Configurar `Account Lockout Threshold` (ex: 5 tentativas) / Set an `Account Lockout Threshold` (e.g., 5 attempts) |
| Senha mínima de 5 caracteres, sem exigência de complexidade / 5-character minimum password, no complexity required | Política de senha forte (mínimo 12+ caracteres, complexidade obrigatória) / Strong password policy (12+ characters minimum, mandatory complexity) |
| Sessão NULL/anônima permitida no SMB / NULL/anonymous session allowed on SMB | Desabilitar sessões anônimas (`restrict anonymous` no Samba) / Disable anonymous sessions (`restrict anonymous` in Samba) |
| Login HTTP sem limite de tentativas (DVWA) / HTTP login with no attempt limit (DVWA) | Rate limiting, CAPTCHA após N tentativas, bloqueio temporário de IP / Rate limiting, CAPTCHA after N attempts, temporary IP blocking |
| Senhas fracas e previsíveis nas wordlists testadas / Weak, predictable passwords in the tested wordlists | Treinamento de usuários, gerenciador de senhas, MFA/2FA / User training, password manager, MFA/2FA |
| Compartilhamento SMB listável sem autenticação / SMB share listable without authentication | Restringir permissões de share, exigir autenticação / Restrict share permissions, require authentication |

## Aprendizados / Key Takeaways

🇧🇷
- Ferramentas de automação como o Medusa são tão boas quanto os parâmetros que você passa para elas — um erro de sintaxe no shell pode gerar resultados falsos positivos silenciosos, por isso validar manualmente ao menos uma credencial "encontrada" é essencial antes de confiar no resultado.
- Enumeração (reconhecimento) antes do ataque de força bruta reduz drasticamente o espaço de busca: em vez de adivinhar usuários, o `enum4linux` revelou contas reais via sessão NULL do SMB.
- Políticas de senha fracas e ausência de lockout são, na prática, o que torna um ataque de força bruta viável — a ferramenta é só o motor, a vulnerabilidade real está na configuração do alvo.
- A sintaxe do módulo HTTP do Medusa é sensível a detalhes pequenos (aspas a mais, `&` faltando entre campos, até um espaço solto entre a flag e o valor). Três erros de digitação diferentes no mesmo comando geraram o mesmo sintoma — todas as credenciais testadas aparecendo como `SUCCESS` — o que reforça a importância de sempre validar manualmente antes de confiar no resultado.
- Credenciais padrão reaproveitadas entre serviços (`msfadmin:msfadmin` válida tanto em FTP quanto em SMB) são um risco real em ambientes corporativos: comprometer um serviço pode abrir caminho para outros.

🇺🇸
- Automation tools like Medusa are only as good as the parameters you give them — a shell syntax error can produce silent false-positive results, which is why manually validating at least one "found" credential is essential before trusting the output.
- Enumeration (reconnaissance) before a brute-force attack drastically narrows the search space: instead of guessing usernames, `enum4linux` revealed real accounts via an SMB NULL session.
- Weak password policies and the absence of account lockout are, in practice, what makes a brute-force attack feasible — the tool is just the engine; the real vulnerability lies in the target's configuration.
- Medusa's HTTP module syntax is sensitive to small details (extra quotes, a missing `&` between fields, even a stray space between a flag and its value). Three different typos in the same command produced the same symptom — every credential tested showing up as `SUCCESS` — reinforcing the importance of always validating manually before trusting the output.
- Default credentials reused across services (`msfadmin:msfadmin` valid on both FTP and SMB) are a real risk in corporate environments: compromising one service can open the door to others.

---

*Projeto desenvolvido para fins educacionais, como parte do desafio "Explorando Vulnerabilidades com Kali Linux e Medusa" da [DIO](https://www.dio.me/). | Project developed for educational purposes, as part of the "Exploring Vulnerabilities with Kali Linux and Medusa" DIO challenge.*
