# Módulo 3 — PoC: Segurança na Camada de Aplicação (Gestão de Sessão)

**Projeto Conexões Seguras | SENAI Felix Guisard**

---

## Objetivo

Demonstrar, através de um servidor web local, a diferença crítica entre cookies de sessão configurados de forma **insegura** e **segura**. A PoC evidencia o risco de **Session Hijacking** (Roubo de Sessão) quando as flags de segurança são omitidas, e como as diretivas `Secure` e `HttpOnly` mitigam esse vetor de ataque.

---

## Ferramentas Usadas

| Ferramenta | Função |
|------------|--------|
| `Node.js` | Ambiente de execução JavaScript no servidor |
| `Express.js` | Framework web para criação do servidor HTTP |
| `curl` | Inspeção dos cabeçalhos HTTP retornados pelo servidor |

---

## Ambiente

- **Plataforma:** AWS Cloud9 / EC2
- **Sistema Operacional:** Ubuntu Server 22.04 LTS

---

## Passo a Passo

Todos os comandos devem ser executados no terminal do AWS Cloud9.

### Etapa 1 — Preparação do Ambiente

Crie o diretório do projeto e instale as dependências:

```bash
# Cria o diretório e entra nele
mkdir modulo3-sessao && cd modulo3-sessao

# Inicializa o projeto Node.js e instala o Express
npm init -y
npm install express
```

### Etapa 2 — Criação do Servidor Web

Crie o arquivo `server.js`:

```bash
touch server.js
```

Abra o arquivo no editor e adicione o seguinte código:

```javascript
const express = require('express');
const app = express();

// Endpoint vulnerável: simula uma aplicação sem diretivas de segurança
app.get('/login-inseguro', (req, res) => {
    // Cookie sem flags de proteção — vulnerável a sniffing (Módulo 1) e XSS
    res.cookie('SessionID', 'abc123vulneravel', { httpOnly: false });
    res.send('Autenticado. Cookie inseguro gerado!');
});

// Endpoint seguro: aplica as melhores práticas de desenvolvimento
app.get('/login-seguro', (req, res) => {
    // 'secure: true'   → cookie só trafega por HTTPS (TLS obrigatório)
    // 'httpOnly: true' → cookie inacessível via JavaScript (proteção XSS)
    res.cookie('SessionID', 'xyz987protegido', { secure: true, httpOnly: true });
    res.send('Autenticado. Diretivas de segurança aplicadas!');
});

app.listen(3000, () => console.log('Servidor de teste ouvindo na porta 3000...'));
```

### Etapa 3 — Inicialização do Servidor

No terminal, execute:

```bash
node server.js
```

O terminal ficará bloqueado exibindo:
```
Servidor de teste ouvindo na porta 3000...
```

**Mantenha este terminal aberto.**

### Etapa 4 — Análise dos Cabeçalhos HTTP

Abra um **novo terminal** e execute os testes com `curl -v` (modo verbose, que exibe todos os cabeçalhos):

**Teste 1 — Endpoint Inseguro:**
```bash
curl -v http://localhost:3000/login-inseguro
```

**Teste 2 — Endpoint Seguro:**
```bash
curl -v http://localhost:3000/login-seguro
```

---

## Resultado

Nas linhas que começam com `< Set-Cookie:`, observe a diferença entre os dois endpoints:

**Teste 1 (inseguro):**
```
< Set-Cookie: SessionID=abc123vulneravel; Path=/
```
O cookie é transmitido **sem qualquer restrição** — exposto a qualquer interceptador de rede e a scripts JavaScript da página.

**Teste 2 (seguro):**
```
< Set-Cookie: SessionID=xyz987protegido; Path=/; HttpOnly; Secure
```
O cookie possui as flags `HttpOnly` e `Secure` ativadas — bloqueado para transmissão em HTTP puro e inacessível via JavaScript.

> Evidências visuais disponíveis em [`../evidencias/cookiesm3.png`](../evidencias/cookiesm3.png), [`../evidencias/iniciarm3.png`](../evidencias/iniciarm3.png) e [`../evidencias/serverm3.png`](../evidencias/serverm3.png).

---

## Explicação Técnica do Resultado

### O que é o Cookie SessionID e quais são os riscos?

O protocolo HTTP é **stateless** (sem estado) — cada requisição é independente, e o servidor não tem memória de requisições anteriores. Para manter sessões autenticadas (saber que o usuário já fez login), as aplicações web utilizam o **Cookie SessionID**: um identificador único e aleatório gerado pelo servidor após o login, armazenado no navegador e enviado automaticamente em toda requisição subsequente.

Para o desenvolvedor, o SessionID representa o risco crítico de **Session Hijacking**:

| Vetor de Ataque | Descrição | Mitigação |
|-----------------|-----------|-----------|
| **Sniffing de rede** | Se o cookie trafegar via HTTP puro (sem TLS), um atacante com modo promíscuo ativo (Módulo 1) captura o SessionID em trânsito e assume a conta sem precisar da senha | Flag `Secure` + HTTPS obrigatório |
| **Cross-Site Scripting (XSS)** | Um script malicioso injetado na página acessa `document.cookie` e exfiltra o SessionID para o servidor do atacante | Flag `HttpOnly` |
| **SessionID previsível** | IDs sequenciais ou baseados em timestamp permitem enumeração/força bruta | Geração com CSPRNG (ex.: `crypto.randomBytes`) |

### O Papel do HTTPS na Proteção de Cookies

A flag `Secure` no cookie instrui o navegador a **nunca transmitir o cookie em conexões HTTP**. Ele só será incluído na requisição quando houver um canal TLS estabelecido — exatamente o mecanismo demonstrado no Módulo 2.

**O que acontece sem HTTPS:**
1. O usuário faz login → servidor gera SessionID e envia via `Set-Cookie`
2. O cookie é transmitido em texto claro em todas as requisições seguintes
3. Um atacante na mesma rede, com `tcpdump` em modo promíscuo, captura o SessionID passivamente
4. O atacante injeta o SessionID capturado no próprio navegador
5. O servidor aceita o cookie como válido → conta comprometida, sem necessidade de credenciais

A combinação **HTTPS + flag `Secure` + flag `HttpOnly`** cria uma **defesa em profundidade**: HTTPS protege o canal de rede, `Secure` garante que o cookie não vaze por engano em HTTP, e `HttpOnly` elimina o vetor de roubo via XSS.
