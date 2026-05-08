# Módulo 2 — PoC: Criptografia Híbrida e Handshake TLS

**Projeto Conexões Seguras | SENAI Felix Guisard**

---

## Objetivo

Simular, por linha de comando, o processo de **Handshake TLS** e a transferência de dados com **Criptografia Híbrida** — o mesmo mecanismo que fundamenta o protocolo HTTPS. A PoC demonstra por que o uso exclusivo de criptografia assimétrica (RSA) para todo o tráfego é inviável, e como a combinação com criptografia simétrica (AES) resolve esse problema.

---

## Ferramenta Usada

| Ferramenta | Função |
|------------|--------|
| `OpenSSL` | Biblioteca e kit de ferramentas para criptografia em linha de comando |

O OpenSSL é o padrão de facto para implementação de SSL/TLS na internet, disponível nativamente no Ubuntu.

---

## Ambiente

- **Plataforma:** AWS Cloud9 / EC2
- **Sistema Operacional:** Ubuntu Server 22.04 LTS

---

## Passo a Passo

Todos os comandos devem ser executados no terminal do AWS Cloud9.

### Etapa 1 — Verificação e Instalação

Certifique-se de que o OpenSSL está disponível:

```bash
sudo apt update && sudo apt install openssl -y
openssl version
```

### Etapa 2 — Geração do Par de Chaves Assimétricas (Simulação do Servidor)

O servidor gera seu par de chaves RSA. A **chave privada** permanece exclusivamente no servidor; a **chave pública** é disponibilizada para qualquer cliente.

```bash
# Gera a chave privada RSA de 2048 bits
openssl genpkey -algorithm RSA -out private_key.pem -pkeyopt rsa_keygen_bits:2048

# Extrai a chave pública a partir da chave privada
openssl rsa -pubout -in private_key.pem -out public_key.pem
```

> **Analogia:** A chave pública funciona como um cadeado aberto que qualquer pessoa pode usar para fechar (criptografar). Apenas quem tem a chave privada consegue abrir (descriptografar).

### Etapa 3 — Geração da Chave Simétrica de Sessão (Simulação do Cliente)

O cliente gera uma chave de sessão AES-256 **única e descartável**. Esta será a chave usada para criptografar os dados reais.

```bash
# Gera uma chave simétrica aleatória de 32 bytes (AES-256)
openssl rand -base64 32 > symmetric_key.bin
```

> A criptografia assimétrica (RSA) é matematicamente pesada demais para criptografar grandes volumes de dados. A solução é usá-la apenas para proteger esta chave de sessão pequena.

### Etapa 4 — Simulação do Handshake (Troca Segura de Chaves)

O cliente criptografa a chave simétrica usando a **chave pública RSA do servidor**. A partir deste momento, somente o servidor (detentor da chave privada) consegue revelar a chave de sessão.

```bash
# Criptografa a chave simétrica com a chave pública RSA
openssl pkeyutl -encrypt -pubin -inkey public_key.pem \
  -in symmetric_key.bin -out encrypted_symmetric.bin
```

> Este é o equivalente ao **Client Key Exchange** no Handshake TLS real.

### Etapa 5 — Transferência de Dados Criptografados

O cliente criptografa o payload de dados usando a chave simétrica AES-256 — muito mais eficiente para grandes volumes.

```bash
# Cria um arquivo com dados sensíveis
echo "Dados corporativos extremamente confidenciais - Modulo 2" > secret.txt

# Criptografa com AES-256-CBC usando a chave simétrica
openssl enc -aes-256-cbc -salt -pbkdf2 \
  -in secret.txt -out secret.enc \
  -pass file:./symmetric_key.bin
```

### Etapa 6 — Descriptografia pelo Servidor (Recepção)

O servidor recebe os dois arquivos criptografados (`encrypted_symmetric.bin` e `secret.enc`) e realiza a descriptografia em duas etapas:

```bash
# 1. Servidor usa a chave privada para revelar a chave simétrica de sessão
openssl pkeyutl -decrypt -inkey private_key.pem \
  -in encrypted_symmetric.bin -out decrypted_symmetric.bin

# 2. Usa a chave simétrica recuperada para descriptografar os dados
openssl enc -d -aes-256-cbc -pbkdf2 \
  -in secret.enc -out decrypted_secret.txt \
  -pass file:./decrypted_symmetric.bin

# 3. Exibe o conteúdo original
cat decrypted_secret.txt
```

---

## Resultado

O comando `cat decrypted_secret.txt` exibe exatamente a string original:

```
Dados corporativos extremamente confidenciais - Modulo 2
```

Durante todo o processo de transferência, os arquivos intermediários (`encrypted_symmetric.bin` e `secret.enc`) permaneceram em **formato binário ilegível** — inúteis para qualquer interceptador sem acesso à chave privada do servidor.

> Evidência visual disponível em [`../evidencias/comandosm2.png`](../evidencias/comandosm2.png).

---

## Explicação Técnica do Resultado

### Por que o HTTPS não usa apenas RSA para todo o tráfego?

A criptografia assimétrica (RSA/ECC) se baseia em operações matemáticas computacionalmente custosas — no caso do RSA, a fatoração de números primos de centenas de dígitos. Para uma conexão HTTPS típica transferindo megabytes de dados (imagens, vídeos, APIs), o uso exclusivo de RSA resultaria em:

- **Latência inaceitável:** operações RSA são centenas de vezes mais lentas que AES equivalente
- **Sobrecarga de CPU:** tanto servidores quanto clientes entrariam em colapso sob tráfego real
- **Escalabilidade zero:** impossível manter milhares de conexões simultâneas

### A Solução: Arquitetura Híbrida

O TLS resolve o problema usando cada algoritmo para sua **força específica**:

| Fase | Algoritmo | Propósito |
|------|-----------|-----------|
| **Handshake** | RSA ou ECDHE (assimétrico) | Trocar a chave de sessão com segurança pela rede insegura |
| **Transferência** | AES-256 (simétrico) | Criptografar o volume de dados em alta velocidade |

O AES possui **aceleração nativa em hardware** (instruções AES-NI nos processadores modernos), o que o torna praticamente sem overhead em relação à transmissão sem criptografia. Um processador moderno consegue criptografar/descriptografar vários GB/s com AES-256.

### Relação com a PoC

- **Etapa 4 = Handshake TLS:** uso de RSA apenas para proteger a troca da chave simétrica
- **Etapa 5 = Application Data:** uso de AES para o dado real, com custo computacional mínimo
- **Arquivos `.enc` e `.bin`:** equivalentes ao tráfego cifrado que um atacante interceptaria — completamente ilegíveis sem a chave privada
