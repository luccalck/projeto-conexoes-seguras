# Módulo 4 — PoC: Tunelamento Seguro com WireGuard (VPN)

**Projeto Conexões Seguras | SENAI Felix Guisard**

---

## Objetivo

Demonstrar a criação de um **túnel VPN criptografado com WireGuard**, evidenciando como o tráfego encapsulado torna o payload ilegível para qualquer interceptador externo — mesmo que este utilize as técnicas de sniffing demonstradas no Módulo 1. A PoC simula um cenário cliente-servidor em uma única máquina utilizando interfaces de rede virtuais (`wg0`).

---

## Ferramenta Usada

| Ferramenta | Função |
|------------|--------|
| `WireGuard` | Protocolo VPN moderno baseado em criptografia Curve25519 e ChaCha20-Poly1305 |
| `iproute2` (`ip`) | Configuração de interfaces de rede e roteamento |
| `tcpdump` | Verificação do tráfego encapsulado no túnel |

### Por que WireGuard?

WireGuard é uma VPN moderna com base de código extremamente enxuta (~4.000 linhas, contra ~100.000 do OpenVPN), integrada ao kernel Linux desde a versão 5.6. Utiliza primitivas criptográficas modernas e de alto desempenho:

| Primitiva | Algoritmo |
|-----------|-----------|
| Troca de chaves | Curve25519 (ECDH) |
| Criptografia de dados | ChaCha20-Poly1305 |
| Hashing | BLAKE2s |
| Derivação de chaves | HKDF |

---

## Ambiente

- **Plataforma:** AWS Cloud9 / EC2
- **Sistema Operacional:** Ubuntu Server 22.04 LTS

---

## Passo a Passo

### Etapa 1 — Instalação do WireGuard

```bash
sudo apt update && sudo apt install wireguard wireguard-tools -y
```

Verifique a instalação:

```bash
wg --version
```

### Etapa 2 — Geração dos Pares de Chaves

Cada peer (servidor e cliente) precisa de um par de chaves próprio. O WireGuard usa **Curve25519** para troca de chaves.

```bash
# Gera o par de chaves do "servidor" (peer A)
wg genkey | tee server_private.key | wg pubkey > server_public.key

# Gera o par de chaves do "cliente" (peer B)
wg genkey | tee client_private.key | wg pubkey > client_public.key

# Exibe as chaves geradas
echo "=== Chave Privada Servidor ===" && cat server_private.key
echo "=== Chave Pública Servidor ===" && cat server_public.key
echo "=== Chave Privada Cliente ===" && cat client_private.key
echo "=== Chave Pública Cliente ===" && cat client_public.key
```

> As chaves são pares de 256 bits codificados em Base64. A chave privada nunca deve ser compartilhada.

### Etapa 3 — Criação da Interface do Servidor (Peer A)

Crie o arquivo de configuração do servidor:

```bash
# Armazena as chaves em variáveis para uso nos comandos
SERVER_PRIVATE=$(cat server_private.key)
CLIENT_PUBLIC=$(cat client_public.key)

# Cria o arquivo de configuração do servidor
sudo bash -c "cat > /etc/wireguard/wg0.conf << EOF
[Interface]
Address = 10.0.0.1/24
PrivateKey = ${SERVER_PRIVATE}
ListenPort = 51820

[Peer]
PublicKey = ${CLIENT_PUBLIC}
AllowedIPs = 10.0.0.2/32
EOF"
```

Suba a interface do servidor:

```bash
sudo wg-quick up wg0
```

### Etapa 4 — Criação da Interface do Cliente (Peer B)

```bash
CLIENT_PRIVATE=$(cat client_private.key)
SERVER_PUBLIC=$(cat server_public.key)

# Cria o arquivo de configuração do cliente
sudo bash -c "cat > /etc/wireguard/wg1.conf << EOF
[Interface]
Address = 10.0.0.2/24
PrivateKey = ${CLIENT_PRIVATE}

[Peer]
PublicKey = ${SERVER_PUBLIC}
AllowedIPs = 10.0.0.1/32
Endpoint = 127.0.0.1:51820
EOF"

# Sobe a interface do cliente
sudo ip link add dev wg1 type wireguard
sudo ip address add dev wg1 10.0.0.2/24
sudo wg setconf wg1 /etc/wireguard/wg1.conf
sudo ip link set up dev wg1
```

### Etapa 5 — Verificação do Status do Túnel

```bash
sudo wg show
```

A saída mostrará as interfaces ativas, os peers configurados e as chaves públicas:

```
interface: wg0
  public key: <SERVER_PUBLIC_KEY>
  private key: (hidden)
  listening port: 51820

peer: <CLIENT_PUBLIC_KEY>
  allowed ips: 10.0.0.2/32
```

### Etapa 6 — Teste de Conectividade pelo Túnel

```bash
# Testa a conectividade através do túnel VPN
ping -c 4 10.0.0.1
```

### Etapa 7 — Captura do Tráfego Encapsulado (Evidência de Criptografia)

Em um terminal secundário, capture o tráfego na interface física (`ens5`) durante o ping pelo túnel:

```bash
# Captura pacotes UDP na porta 51820 (protocolo WireGuard)
sudo tcpdump -i ens5 -nn -A udp port 51820
```

### Etapa 8 — Encerramento e Limpeza

```bash
sudo wg-quick down wg0
sudo ip link del dev wg1
```

---

## Resultado

**Ping pelo túnel:**
```
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=0.123 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=0.098 ms
```
A conectividade entre os peers pelo túnel VPN é confirmada.

**Captura com tcpdump na interface física:**
```
12:34:56.789012 IP 127.0.0.1.51820 > 127.0.0.1.XXXXX: UDP, length 128
E...<...@......
............ [PAYLOAD BINÁRIO ILEGÍVEL] .#...m..Z.!..f..
```
O payload capturado na interface física é **binário e completamente ilegível** — o conteúdo real do ping está encapsulado e criptografado com **ChaCha20-Poly1305** dentro dos pacotes UDP do WireGuard.

---

## Explicação Técnica do Resultado

### O que é uma VPN e como o WireGuard funciona?

Uma **VPN (Virtual Private Network)** cria um túnel lógico criptografado entre dois ou mais endpoints, permitindo que tráfego sensível trafegue por redes não confiáveis (como a internet) sem exposição do conteúdo.

O WireGuard opera na **Camada 3 (Rede)** como um driver de kernel, criando interfaces de rede virtuais (ex.: `wg0`). Todo pacote IP endereçado a um peer configurado é automaticamente:

1. **Encapsulado** em um datagrama UDP
2. **Criptografado** com ChaCha20-Poly1305 usando a chave de sessão derivada via Curve25519
3. **Transmitido** pela interface física para o endpoint do peer

No destino, o processo inverso ocorre: o WireGuard descriptografa o pacote UDP e entrega o pacote IP original à pilha de rede.

### Relação com os Módulos Anteriores

| Técnica do Atacante | Resultado sem VPN | Resultado com VPN |
|---------------------|-------------------|-------------------|
| Sniffing (Módulo 1) | Dados HTTP legíveis | Payload binário ilegível |
| Sem TLS (Módulo 2) | Texto claro exposto | Criptografado na camada de rede |
| Cookie SessionID (Módulo 3) | Interceptável em HTTP | Canal protegido pelo túnel |

A VPN complementa o TLS adicionando uma camada adicional de proteção no nível da rede — mesmo que uma aplicação esqueça de usar HTTPS, o tráfego dentro do túnel WireGuard permanece criptografado para observadores externos.

### Vantagem do WireGuard sobre IPSec/OpenVPN

O WireGuard possui **latência de handshake inferior a 1 round-trip** (contra múltiplos rounds do IKEv2/IPSec) e a base de código minimalista facilita auditorias de segurança e reduz a superfície de ataque.
