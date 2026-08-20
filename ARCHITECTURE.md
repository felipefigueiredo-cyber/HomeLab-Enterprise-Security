# 🗺️ Arquitetura de Redes MZSD (Multi-Zone Segmented Defense)

Este documento detalha as especificações técnicas de redes, endereçamentos IP, regras de firewall e fluxos de dados de segurança implementados no projeto **Enterprise Security Sandbox**.

---

## 1. Topologia de Rede Virtualizada

A arquitetura foi implementada no hipervisor VirtualBox utilizando o conceito de **Redes Internas** para simular switches lógicos isolados por VLANs. O firewall pfSense atua como o único roteador de trânsito (Gateway) na topologia *Router-on-a-Stick*.

![Arquitetura MZSD](arquitetura_mzsd.png)

---

## 2. Matriz de Endereçamento IP

Para garantir a identificação estrita dos ativos da rede, a infraestrutura foi mapeada conforme a tabela abaixo:

| Dispositivo / VM | Interface de Rede | Tipo de IP | Endereço IP | Máscara de Rede | Gateway Padrão |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VM 01 - pfSense Firewall** | WAN | Dinâmico (DHCP) | *IP do Host (NAT)* | `255.255.255.0` | IP do Host |
| **VM 01 - pfSense Firewall** | LAN (VLAN 10) | Estático | **`192.168.10.1`** | `255.255.255.0` (/24) | *Não aplicável* |
| **VM 01 - pfSense Firewall** | MGMT (VLAN 99) | Estático | **`192.168.99.1`** | `255.255.255.0` (/24) | *Não aplicável* |
| **VM 02 - FreeRADIUS** | LAN (VLAN 99) | Estático | **`192.168.99.10`** | `255.255.255.0` (/24) | `192.168.99.1` |
| **VM 03 - Wazuh SIEM** | LAN (VLAN 99) | Estático | **`192.168.99.20`** | `255.255.255.0` (/24) | `192.168.99.1` |
| **VM 04 - Lubuntu PAW** | LAN (VLAN 99) | Estático | **`192.168.99.100`**| `255.255.255.0` (/24) | `192.168.99.1` |
| **VM 05 - Lubuntu Cliente** | LAN (VLAN 10) | Dinâmico (DHCP)| **`192.168.10.102`**| `255.255.255.0` (/24) | `192.168.10.1` |

---

## 🔒 3. Matriz de Portas do Firewall & Fluxo de Dados

O pfSense aplica o princípio de **Bloqueio por Padrão (Default Deny)**. A comunicação entre os segmentos de rede só é permitida através dos guichês (portas) e protocolos de segurança explicitados abaixo:

| Sentido do Fluxo (Origem -> Destino) | Protocolo | Porta de Destino | Serviço | Objetivo de Segurança |
| :--- | :--- | :--- | :--- | :--- |
| **VLAN 10 (LAN) -> pfSense LAN** | UDP | `1812` / `1813` | RADIUS Auth / Acct | Autenticar o computador do funcionário via Captive Portal. |
| **VLAN 99 (MGMT) -> pfSense MGMT**| TCP | `443` (HTTPS) | WebConfigurator | Permitir que **apenas** a PAW administre o firewall via navegador. |
| **VM 02 & VM 03 -> pfSense MGMT** | UDP | `123` | NTP (Tempo) | Sincronizar os relógios dos servidores de forma segura com o NTP mestre do pfSense. |
| **VM 02 & VM 04 -> Wazuh SIEM** | TCP | `1514` / `1515` | Wazuh Agent API | Enviar logs criptografados de auditoria local para a central do SOC. |
| **pfSense (VM 01) -> Wazuh SIEM** | UDP | `514` | Syslog Remoto | Enviar logs de firewall, bloqueios e eventos do pfSense para o SIEM. |
| **VLAN 10 (LAN) -> Internet** | TCP/UDP| `80` (HTTP) / `443` (HTTPS)| Navegação Web | Permitir saída controlada para a internet (após autenticação no RADIUS). |
| **VLAN 99 (MGMT) -> Internet** | *Bloqueado*| *Bloqueado* | *Bloqueado* | **Totalmente bloqueado por padrão (Hardening).** Os servidores e a PAW não possuem saída para a rede externa. |

---

## 🛡️ 4. Isolamento Lógico (Regras de Bloqueio Ativas)

Para conter possíveis ameaças e evitar movimentos laterais na rede (Defesa em Profundidade), as seguintes regras de descarte de pacotes estão ativas:
*   **Bloqueio Total VLAN 10 -> VLAN 99:** Qualquer pacote de dados vindo da rede de usuários (VLAN 10) tentando acessar qualquer IP da rede de gerência (VLAN 99) é sumariamente descartado pelo pfSense.
*   **Bloqueio de Gerência WAN:** O acesso à interface web administrativa do pfSense através do IP da internet (WAN) é totalmente bloqueado. O gerenciamento só é aceito vindo da interface local segura MGMT (VLAN 99).
