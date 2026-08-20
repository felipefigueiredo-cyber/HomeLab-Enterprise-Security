# 🗺️ Arquitetura de Redes MZSD (Multi-Zone Segmented Defense)

Este documento detalha as especificações técnicas de redes, mapeamentos de endereçamento IP, regras de controle de acesso (Firewall) e fluxos lógicos de dados implementados no projeto *Enterprise Security Sandbox*.

---

## 1. Topologia de Rede Virtualizada

A arquitetura foi totalmente implementada dentro do hipervisor VirtualBox, utilizando o conceito de **Redes Internas** (Internal Networks) para simular switches lógicos isolados por VLANs. 

O firewall pfSense atua como o único roteador de trânsito (Gateway padrão) para todos os segmentos, implementando uma topologia robusta e isolada de tráfego inter-VLAN.

![Arquitetura MZSD](./assets/arquitetura_mzsd.png)

---

## 2. Matriz de Endereçamento IP

Para garantir a identificação e o rastreamento estrito dos ativos de rede, a infraestrutura foi mapeada conforme a distribuição abaixo:

| Dispositivo / VM | Interface de Rede | Tipo de IP | Endereço IP | Máscara de Rede | Gateway Padrão |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VM 01 - pfSense Firewall** | WAN (`em0`) | Dinâmico (DHCP) | IP do Host (NAT) | `255.255.255.0` | IP do Host |
| **VM 01 - pfSense Firewall** | LAN (`em2` / VLAN 10)| Estático | `192.168.10.1` | `255.255.255.0` (/24) | Não aplicável |
| **VM 01 - pfSense Firewall** | MGMT (`em1` / VLAN 99)| Estático | `192.168.99.1` | `255.255.255.0` (/24) | Não aplicável |
| **VM 02 - FreeRADIUS** | MGMT (VLAN 99) | Estático | `192.168.99.10` | `255.255.255.0` (/24) | `192.168.99.1` |
| **VM 03 - Wazuh SIEM** | MGMT (VLAN 99) | Estático | `192.168.99.20` | `255.255.255.0` (/24) | `192.168.99.1` |
| **VM 04 - Lubuntu PAW** | MGMT (VLAN 99) | Estático | `192.168.99.100` | `255.255.255.0` (/24) | `192.168.99.1` |
| **VM 05 - Lubuntu Cliente** | LAN (VLAN 10) | Dinâmico (DHCP) | `192.168.10.102` | `255.255.255.0` (/24) | `192.168.10.1` |

---

## 🔒 3. Matriz de Portas do Firewall & Fluxo de Dados

O pfSense aplica o princípio rígido de **Bloqueio por Padrão** (*Default Deny*). Toda a comunicação entre segmentos de rede só é permitida através das portas, protocolos e origens estritamente especificadas na matriz de segurança abaixo:

| Sentido do Fluxo (Origem -> Destino) | Protocolo | Porta de Destino | Serviço | Objetivo de Segurança |
| :--- | :--- | :--- | :--- | :--- |
| **VLAN 10 (LAN) -> pfSense LAN** | TCP | `8002` | Captive Portal | Usuário comum é redirecionado para a página de autenticação corporativa. |
| **pfSense (VM 01) -> VM 02 (FreeRADIUS)**| UDP | `1812` / `1813` | RADIUS Auth / Acct | O gateway consulta as credenciais do Captive Portal e audita as sessões ativas. |
| **VM 04 (PAW) -> pfSense MGMT** | TCP | `443` (HTTPS) | WebConfigurator | **Privilégio Mínimo:** Apenas o IP estático da PAW possui permissão para administrar o firewall via navegador. |
| **VLAN 99 (MGMT) -> pfSense MGMT** | UDP | `123` | NTP (Tempo) | Sincronismo de relógios dos servidores com o servidor NTP mestre local do pfSense. |
| **VM 02 & VM 04 -> VM 03 (Wazuh SIEM)**| TCP | `1514` / `1515` | Wazuh Agent Connection | Envio seguro e criptografado de logs de eventos e alertas para o centralizador do SOC. |
| **pfSense (VM 01) -> VM 03 (Wazuh SIEM)**| UDP | `514` | Syslog Remoto | Exportação de logs de tráfego, conexões e bloqueios do firewall para correlação no SIEM. |
| **VLAN 10 (LAN) -> Internet (WAN)** | TCP/UDP | `80` (HTTP) / `443` (HTTPS) | Navegação Web | Permissão de tráfego de saída controlado para a internet (liberado apenas após login). |
| **VLAN 99 (MGMT) -> Internet (WAN)** | Bloqueado | Bloqueado | Bloqueado | **Hardening:** Totalmente bloqueado. Servidores críticos e a máquina administrativa não possuem saída para a internet. |
| **VLAN 10 (LAN) -> pfSense LAN** | TCP | `443` (HTTPS) | WebConfigurator | **Hardening:** Impede terminantemente que usuários comuns acessem ou façam força bruta na tela de gerência do pfSense. |

---

## 🛡️ 4. Isolamento Lógico (Regras de Bloqueio Ativas)

Para conter possíveis tentativas de invasão e evitar o movimento lateral na rede (seguindo os conceitos de *Defesa em Profundidade*), as seguintes políticas ativas foram implementadas no firewall:

1.  **Bloqueio Total Inter-VLAN (VLAN 10 -> VLAN 99):** Qualquer pacote de dados iniciado na rede de usuários (VLAN 10) que tente alcançar qualquer servidor ou ativo da rede de gerenciamento (VLAN 99) é sumariamente descartado pelo pfSense.
2.  **Acesso Administrativo de Confiança Zero (Zero Trust):**
    *   O acesso ao console de gerenciamento web do pfSense (WebConfigurator) a partir da interface WAN (Internet) é totalmente desativado.
    *   Na rede MGMT (VLAN 99), aplicando o princípio de privilégio mínimo em nível de host, o acesso administrativo (HTTPS na porta `443` e SSH na porta `22`) do pfSense foi restrito para aceitar conexões vindas **exclusivamente** do IP estático `192.168.99.100` (VM 04 - Lubuntu PAW). 
    *   Qualquer outra máquina conectada à VLAN 99 (incluindo os servidores FreeRADIUS e Wazuh) terá o tráfego bloqueado se tentar interagir com as portas administrativas do gateway, mitigando riscos em caso de comprometimento de um dos servidores.
