# Projeto Conexões Seguras

**Faculdade de Tecnologia SENAI Felix Guisard**  
Curso Superior de Tecnologia em Análise e Desenvolvimento de Sistemas  
Projeto Semestral — Parte 1/3

---

## Visão Geral

Este repositório contém a documentação técnica e as Provas de Conceito (PoCs) desenvolvidas para o projeto **Conexões Seguras**, cobrindo os principais mecanismos de segurança em redes corporativas modernas.

O projeto é dividido em quatro módulos práticos, cada um abordando uma camada distinta da pilha de segurança — da interceptação de tráfego em nível de rede até a proteção de sessões na camada de aplicação.

## Estrutura do Repositório

```
projeto-conexoes-seguras/
│
├── modulo1-sniffing/     # PoC: Sniffing e Interceptação (Camada 2 e 3)
├── modulo2-tls/          # PoC: Criptografia Híbrida e Handshake TLS
├── modulo3-sessao/       # PoC: Segurança na Camada de Aplicação (Sessões)
├── modulo4-vpn/          # PoC: Tunelamento Seguro com WireGuard (VPN)
│
├── evidencias/           # Screenshots e capturas de saída dos experimentos
├── relatorio.md          # Relatório Técnico completo (≤ 5 páginas)
└── README.md             # Este arquivo
```

## Módulos

| Módulo | Tema | Ferramentas |
|--------|------|-------------|
| [Módulo 1](modulo1-sniffing/README.md) | Sniffing e Interceptação | `tcpdump`, `iproute2`, `curl` |
| [Módulo 2](modulo2-tls/README.md) | Criptografia Híbrida e TLS | `OpenSSL` |
| [Módulo 3](modulo3-sessao/README.md) | Gestão Segura de Sessões | `Node.js`, `Express.js`, `curl` |

## Ambiente de Laboratório

Todos os experimentos foram executados em uma instância **Ubuntu Server 22.04 LTS** provisionada na infraestrutura da AWS (Cloud9/EC2), garantindo um ambiente isolado e reproduzível.

## Entregas

- **Relatório Técnico:** [`relatorio.md`](relatorio.md)
- **Implementações Práticas:** Pasta de cada módulo contém `README.md` com o passo a passo completo
- **Evidências:** Pasta [`evidencias/`](evidencias/) com capturas de saída de cada experimento
- **Documentação Online:** [GitHub Pages](https://luccalck.github.io/projeto-conexoes-seguras)
