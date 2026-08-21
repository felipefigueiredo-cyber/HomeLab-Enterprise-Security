# 🛠️ Guia Técnico de Implantação e Resolução de Problemas (Troubleshooting)

Este documento registra de forma técnica, sequencial e detalhada todos os comandos de terminal, configurações de serviços e procedimentos de resolução de problemas executados durante a implantação da infraestrutura de segurança **MZSD**.

---

## 1. Comandos de Instalação e Configuração dos Serviços

### A. VM 02 - Servidor de Autenticação FreeRADIUS
O servidor foi implantado utilizando o Ubuntu Server LTS (Headless) com o IP estático **`192.168.99.10/24`** configurado manualmente na fase de instalação.

```bash
# Atualizar as listas de repositórios do Ubuntu
sudo apt update

# Instalar o FreeRADIUS e as ferramentas de teste de cliente
sudo apt install -y freeradius freeradius-utils

# Verificar se o serviço iniciou com sucesso
sudo systemctl status freeradius
```

*   **Configuração do Cliente (pfSense) em `/etc/freeradius/3.0/clients.conf`:**
    ```text
    client pfsense {
        ipaddr = 192.168.99.1
        secret = senha_secreta_pfsense
        nas_type = other
    }
    ```
*   **Cadastro do Usuário de Teste em `/etc/freeradius/3.0/users`:**
    ```text
    joao Cleartext-Password := "joao123"
    ```
*   **Comando de Reinicialização para Aplicar as Configurações:**
    ```bash
    sudo systemctl restart freeradius
    ```
*   **Teste de Validação de Autenticação Logins:**
    ```bash
    # Comando radtest simulando tentativa de login local
    radtest joao joao123 127.0.0.1 0 testing123
    
    # Retorno de Sucesso Esperado:
    # Received Access-Accept
    ```

---

### B. VM 03 - Wazuh SIEM
O servidor foi implantado utilizando o Ubuntu Server LTS (4 GB de RAM e 2 vCPUs) com o IP estático **`192.168.99.20/24`** configurado na fase de instalação.

```bash
# Baixar o script oficial do instalador automatizado do Wazuh
curl -sO https://packages.wazuh.com/4.8/wazuh-install.sh

# Executar a instalação unificada ignorando a checagem de versão do Ubuntu 26.04
sudo bash wazuh-install.sh --all-in-one --ignore-check
```

---

### C. Instalação e Ativação do Wazuh Agent (Na PAW e no FreeRADIUS)
O comando gerado no painel do Wazuh foi executado via SSH nas máquinas para registrá-las na central do SOC:

```bash
# Baixar e instalar o pacote do agente apontando para o IP do SIEM (192.168.99.20)
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.8.2-1_amd64.deb && sudo WAZUH_MANAGER='192.168.99.20' dpkg -i ./wazuh-agent_4.8.2-1_amd64.deb

# Recarregar e ativar o serviço do agente no Linux
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

---

### D. Configuração do Sincronismo de Tempo Local (NTP)
Para garantir que as VMs 02 e 03 sincronizem seus relógios de forma segura a partir do pfSense (`192.168.99.1`) via porta `123/UDP`, o serviço de tempo foi configurado em ambas:

```bash
# Instalar o serviço timesyncd (caso esteja ausente por padrão)
sudo apt install -y systemd-timesyncd

# Editar o arquivo de configuração
sudo nano /etc/systemd/timesyncd.conf
```

*   **Linha editada e ativada em `/etc/systemd/timesyncd.conf`:**
    ```text
    NTP=192.168.99.1
    ```
*   **Aplicar as configurações e reiniciar o serviço:**
    ```bash
    sudo systemctl restart systemd-timesyncd
    
    # Comando de verificação de sucesso de contato com o pfSense:
    sudo systemctl status systemd-timesyncd
    # Retorno esperado: Contacted time server 192.168.99.1:123 (192.168.99.1)
    ```
*   **Mudar o Fuso Horário das VMs para Cuiabá (Alinhamento de Logs):**
    ```bash
    sudo timedatectl set-timezone America/Cuiaba
    ```

---

## 🛠️ 2. Relatório de Troubleshooting (Resolução de Desafios Técnicos)

Esta seção registra de forma detalhada as falhas encontradas durante a implantação prática e as soluções aplicadas para contorná-las:

### Desafio 1: Bloqueio de Borda Wi-Fi do Roteador Físico (pfSense WAN)
*   **Sintoma:** O pfSense configurado em modo Bridge (conectado à placa Wi-Fi do host) não conseguia se conectar à internet e travava a instalação básica.
*   **Causa:** O roteador físico residencial bloqueava múltiplos endereços MACs no mesmo canal Wi-Fi por segurança.
*   **Solução:** Alteração da interface WAN do pfSense (Adaptador 1) para **Modo NAT** no VirtualBox. O Windows Host passou a mascarar o tráfego do pfSense sob o seu próprio IP, liberando a navegação de forma estável.

### Desafio 2: Estouro de Armazenamento do Wazuh SIEM (VM 03)
*   **Sintoma:** A página web do Wazuh Dashboard exibia o erro `Wazuh dashboard server is not ready yet` ou `Something went wrong` e o banco de dados `wazuh-indexer` ficava travado infinitamente no status de `activating (start)`.
*   **Análise Forense de Logs:** O comando `sudo tail -n 40 /var/log/wazuh-indexer/wazuh-cluster.log` revelou o erro: `java.io.IOException: No space left on device`. O disco virtual de 25 GB original estava com 100% de uso.
*   **Solução em Três Fases:**
    1.  *Fase Física:* Desligamento da VM 03 e expansão do arquivo virtual `.vdi` de 25 GB para 60 GB no Gerenciador de Mídias Virtuais do VirtualBox.
    2.  *Fase Lógica (Linux):* Ligação da VM e execução do comando **`sudo growpart /dev/sda 2`** (para esticar a tabela de partição ativa) e do comando **`sudo resize2fs /dev/sda2`** (para esticar o sistema de arquivos ext4 online). O uso do disco caiu de 100% para **44%**, liberando 32 GB de espaço livre.
    3.  *Destrancar o Banco:* O estouro de disco havia ativado a trava de segurança de "Somente Leitura" (*Read-Only*) do OpenSearch. O banco de dados foi destrancado enviando um comando direto para a API interna do indexer (aplicando o comando apenas aos índices `wazuh-*` para preservar as pastas de segurança protegidas do sistema):
        ```bash
        curl -XPUT -k -u admin:[SENHA] "https://localhost:9200/wazuh-*/_settings" -H 'Content-Type: application/json' -d'{"index.blocks.read_only_allow_delete": null}'
        ```

### Desafio 3: Exclusão Acidental de Arquivos de Sistema durante Cancelamento de Script
*   **Sintoma:** O comando de alteração de senhas `wazuh-passwords-tool.sh` retornava o erro: `line 67: /var/ossec/bin/wazuh-keystore: No such file or directory` e o serviço `wazuh-manager` não iniciava mais.
*   **Causa:** Um comando de instalação com a tag `--overwrite` foi executado acidentalmente e interrompido no meio do caminho com a opção "N". Porém, antes da confirmação de parada, o script já havia excluído fisicamente os arquivos binários do `wazuh-manager`.
*   **Solução:** Executada uma reinstalação completa e limpa do zero usando o comando `--overwrite` completo. Com o disco agora saudável (60 GB), a instalação foi finalizada com sucesso e os arquivos binários foram integralmente restaurados.

### Desafio 4: Erro de Sincronia de Credenciais do Wazuh Dashboard
*   **Sintoma:** O painel web do Wazuh exibia o erro `Invalid username or password` ao tentar logar com a senha do admin gerada pelo sistema.
*   **Causa:** Devido aos travamentos anteriores do disco cheio, o arquivo de senhas do painel web (`/etc/wazuh-dashboard/wazuh.yml`) não foi atualizado de forma síncrona com a nova senha mestre do banco de dados (Indexer).
*   **Solução:** Executada a ferramenta oficial de redefinição de senhas do Wazuh de forma centralizada para sincronizar todos os arquivos de uma vez só:
    ```bash
    sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin -p 'FelipeSOC123.'
    ```

### Desafio 5: Erro de Sintaxe de Dicionário no FreeRADIUS
*   **Sintoma:** O comando de reiniciar o serviço `sudo systemctl restart freeradius` falhava em vermelho com o erro `control process exited with error code`.
*   **Causa:** O interpretador de dicionário do FreeRADIUS é extremamente rígido com sintaxe. No arquivo `/etc/freeradius/3.0/users`, foi digitado `Cleartexte-password` (com um "e" extra no final do Cleartext e com a letra "p" minúscula), o que gerou erro de sintaxe por não existir no dicionário oficial.
*   **Solução:** Correção da linha de texto para o padrão oficial exigido pelo sistema: `Cleartext-Password` (com o "P" maiúsculo e sem a letra "e" extra), restaurando a inicialização imediata do serviço.

### Desafio 6: Conflito de Duplo IP na VM do Funcionário (VM 05)
*   **Sintoma:** O João (VM 05) se autenticava no Captive Portal com sucesso, mas o navegador continuava sem carregar a internet.
*   **Causa:** A placa de rede `enp0s3` do João estava com dois IPs ativos ao mesmo tempo: o `192.168.10.100` (estático antigo) e o `192.168.10.102` (dinâmico DHCP). O pfSense autorizava o tráfego do IP `.100` no portal, mas a VM tentava navegar usando o IP `.102` que não estava autenticado, gerando o bloqueio pelo firewall.
*   **Solução:** Deletado o perfil estático antigo na janela gráfica de redes e executado o reinício físico da VM 05. A placa de rede ligou limpa com apenas um único IP dinâmico (`192.168.10.102`) do DHCP, estabelecendo a navegação e a autenticação de forma síncrona e perfeita.
