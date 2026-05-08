# Módulo 1 — PoC: Sniffing e Interceptação de Tráfego

**Projeto Conexões Seguras | SENAI Felix Guisard**

---

## Objetivo

Demonstrar, em ambiente controlado, como o **modo promíscuo** de uma placa de rede (NIC) permite a captura de pacotes que não são endereçados ao host local, evidenciando a vulnerabilidade de protocolos sem criptografia (HTTP) frente a ataques de interceptação passiva na Camada 2 (Data Link) e Camada 3 (Rede).

---

## Ferramenta Usada

| Ferramenta | Função |
|------------|--------|
| `iproute2` (`ip`) | Gerenciamento e configuração de interfaces de rede |
| `tcpdump` | Captura e análise de pacotes de rede |
| `curl` | Geração de tráfego HTTP não criptografado |

---

## Ambiente

- **Plataforma:** AWS Cloud9 / EC2
- **Sistema Operacional:** Ubuntu Server 22.04 LTS
- **Tipo de Instância:** t3.micro

### Provisionamento da Instância (AWS Cloud9)

1. Acesse o Console AWS e pesquise por **Cloud9**
2. Clique em **Create environment** e configure:
   - **Name:** `Laboratorio-Conexoes-Seguras`
   - **Environment type:** New EC2 instance
   - **Instance type:** t3.micro
   - **Platform:** Ubuntu Server 22.04 LTS
   - **Timeout:** 30 minutes
3. Aguarde o provisionamento até que o terminal esteja disponível

---

## Passo a Passo

### Etapa 1 — Instalação das Dependências

Atualize os repositórios e instale as ferramentas necessárias:

```bash
sudo apt update && sudo apt install tcpdump curl -y
```

### Etapa 2 — Identificação da Interface de Rede

Liste as interfaces disponíveis e identifique qual está ativa (geralmente `ens5` em instâncias EC2):

```bash
ip link show
```

O nome da interface ativa aparecerá com o status `UP` na saída do comando. Anote-o para os próximos passos (substitua `ens5` pelo nome correto em seu ambiente).

### Etapa 3 — Habilitação do Modo Promíscuo

Ative o modo promíscuo na interface identificada, forçando a NIC a capturar todos os quadros do segmento:

```bash
# Ativar o modo promíscuo
sudo ip link set dev ens5 promisc on

# Validar a ativação (procure pela flag PROMISC na saída)
ip link show ens5
```

**Saída esperada na validação:**
```
2: ens5: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> ...
```

### Etapa 4 — Inicialização do Sniffer

Inicie o `tcpdump` para monitorar o tráfego HTTP (porta 80), exibindo o payload em formato ASCII:

```bash
sudo tcpdump -i ens5 -nn -A -s 0 port 80
```

| Flag | Descrição |
|------|-----------|
| `-i ens5` | Interface a monitorar |
| `-nn` | Não resolve nomes DNS nem portas |
| `-A` | Exibe payload em ASCII |
| `-s 0` | Captura o pacote completo (sem limite de bytes) |
| `port 80` | Filtra apenas tráfego HTTP |

**Mantenha este terminal aberto.**

### Etapa 5 — Injeção de Tráfego (Terminal Secundário)

Abra uma nova aba de terminal e execute uma requisição HTTP não criptografada:

```bash
curl http://neverssl.com
```

> `neverssl.com` é um site que garante nunca usar HTTPS, sendo ideal para testes de sniffing.

### Etapa 6 — Encerramento e Restauração

Após capturar as evidências, desative o modo promíscuo para restaurar o comportamento padrão da NIC:

```bash
sudo ip link set dev ens5 promisc off
```

---

## Resultado

No terminal do `tcpdump`, o tráfego HTTP é exibido **em texto legível imediatamente** após a requisição `curl`. A captura evidencia:

- O cabeçalho da requisição: `GET / HTTP/1.1`
- O cabeçalho `Host:`, `User-Agent:` e demais metadados
- O corpo da resposta contendo as tags HTML do servidor de destino

```
GET / HTTP/1.1
Host: neverssl.com
User-Agent: curl/7.81.0
Accept: */*

HTTP/1.1 200 OK
Content-Type: text/html
...
<html>...</html>
```

> Evidência visual disponível em [`../evidencias/comandom1.png`](../evidencias/comandom1.png) e [`../evidencias/resultadom1.png`](../evidencias/resultadom1.png).

---

## Explicação Técnica do Resultado

### O que é o Modo Promíscuo?

Em operação normal, o controlador de interface de rede (NIC) implementa um **filtro de endereços MAC** no nível do hardware/driver. Apenas quadros Ethernet cujo endereço MAC de destino corresponde ao endereço da própria interface (ou ao MAC de broadcast) são encaminhados para a pilha de rede do sistema operacional. Todos os demais quadros são descartados silenciosamente.

O **modo promíscuo** desabilita esse filtro. A NIC passa a encaminhar **todos os quadros interceptados no segmento de rede** diretamente para a CPU, independentemente do endereço MAC de destino. Isso transforma a máquina em um analisador passivo de tráfego — qualquer host no mesmo segmento pode ter seu tráfego capturado.

### Por que o HTTP é Vulnerável?

O protocolo HTTP opera na **Camada de Aplicação** sem nenhuma camada de segurança de transporte (TLS/SSL). Seus dados são transmitidos em **Texto Claro (Plain Text)**, o que significa que:

- Qualquer pacote interceptado pelo modo promíscuo é imediatamente legível
- Credenciais de login, tokens de sessão e dados sensíveis ficam expostos
- Não há como detectar ou prevenir a interceptação passiva

### Implicação para Redes Corporativas

Em redes com switches gerenciados, o ataque clássico requer técnicas adicionais como **ARP Poisoning** para redirecionar o tráfego. Porém, em redes sem fio (Wi-Fi aberto/WPA2-Enterprise com configurações fracas) ou em segmentos hub, o modo promíscuo é suficiente para expor todo o tráfego HTTP do segmento.

A mitigação definitiva é a adoção do **HTTPS (HTTP sobre TLS)**, demonstrado no Módulo 2.
