# 🛠️ Guia Técnico de Implantação e Resolução de Problemas (Troubleshooting)

Este documento registra de forma técnica, sequencial e detalhada todos os comandos de terminal, arquivos de configuração de serviços e procedimentos de resolução de incidentes executados durante a implantação da infraestrutura de segurança do laboratório.

---

## 1. Comandos de Instalação e Configuração dos Serviços

### A. VM 02 - Servidor de Autenticação FreeRADIUS
O servidor de autenticação centralizada foi implantado utilizando o Ubuntu Server LTS (Headless) com o IP estático `192.168.99.10/24` configurado manualmente.

```bash
# Atualizar as listas de repositórios do sistema operacional
sudo apt update

# Instalar o FreeRADIUS e utilitários de teste de cliente
sudo apt install -y freeradius freeradius-utils

# Verificar o status ativo do serviço recém-instalado
sudo systemctl status freeradius
```

*   **Configuração do Cliente pfSense em `/etc/freeradius/3.0/clients.conf`:**
    ```text
    client pfsense_gateway {
        ipaddr = 192.168.99.1
        secret = senha_secreta_pfsense_radius
        nas_type = other
    }
    ```

*   **Cadastro do Usuário de Teste Ativo em `/etc/freeradius/3.0/users`:**
    ```text
    felipe Cleartext-Password := "felipe123"
    ```
    *Nota: O usuário joao não foi incluído intencionalmente neste arquivo para permitir o teste de login malsucedido (Access-Reject).*

*   **Comando de Reinicialização para Aplicar as Novas Configurações:**
    ```bash
    # Reiniciar o serviço para aplicar as novas configurações de clientes e usuários
    sudo systemctl restart freeradius

    # Executar ferramenta de validação local (radtest) simulando uma autenticação
    radtest felipe felipe123 127.0.0.1 0 testing123
    ```

*   **Retorno de Sucesso Esperado:**
    ```text
    Received Access-Accept Id 0 from 127.0.0.1:1812 to 127.0.0.1:0 length 20
    ```

---

### B. VM 03 - Wazuh SIEM
O servidor centralizador do SOC foi implantado utilizando o Ubuntu Server LTS (configurado com 4 GB de RAM, 2 vCPUs e IP estático `192.168.99.20/24`).

```bash
# Realizar o download do script oficial de instalação automatizada do Wazuh
curl -sO https://packages.wazuh.com/4.8/wazuh-install.sh

# Executar a instalação unificada utilizando a tag para ignorar checagem de versão no Ubuntu 26.04
sudo bash wazuh-install.sh --all-in-one --ignore-check
```

---

### C. Instalação e Ativação do Wazuh Agent (PAW e FreeRADIUS)
O agente de coleta de logs do Wazuh foi instalado de forma idêntica em ambos os servidores da rede de gerenciamento para encaminhar os eventos ao SIEM:

```bash
# Baixar o pacote oficial do agente compatível com arquitetura amd64
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.8.2-1_amd64.deb

# Realizar a instalação apontando o agente para o IP do SIEM Manager (192.168.99.20)
sudo WAZUH_MANAGER='192.168.99.20' dpkg -i ./wazuh-agent_4.8.2-1_amd64.deb

# Recarregar as definições do systemd e habilitar o início automático do agente
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

---

### D. Configuração de Sincronismo de Tempo Local (NTP)
Para garantir a precisão milimétrica da marcação temporal de todos os logs coletados (correlação forense), configuramos o pfSense (`192.168.99.1`) como o servidor NTP mestre para as VMs 02 e 03:

```bash
# Garantir a presença do utilitário padrão systemd-timesyncd
sudo apt install -y systemd-timesyncd

# Editar o arquivo de configuração do sincronizador
sudo nano /etc/systemd/timesyncd.conf
```

*   **Parâmetros de sincronização configurados em `/etc/systemd/timesyncd.conf`:**
    ```text
    [Time]
    NTP=192.168.99.1
    ```

```bash
# Aplicar as alterações reiniciando o serviço de sincronismo
sudo systemctl restart systemd-timesyncd

# Validar se o contato com o pfSense foi estabelecido com sucesso na porta 123/UDP
sudo systemctl status systemd-timesyncd
```

*   **Retorno de Sucesso Esperado:**
    ```text
    Contacted time server 192.168.99.1:123 (192.168.99.1).
    ```

*   **Alinhamento do Fuso Horário Local (Cuiabá):**
    ```bash
    sudo timedatectl set-timezone America/Cuiaba
    ```

---

## 🛠️ 2. Relatório de Troubleshooting (Resolução de Desafios Técnicos)

Esta seção documenta os problemas reais identificados durante o processo de implantação prática e as etapas executadas para solucioná-los.

### Desafio 1: Bloqueio de Borda Wi-Fi do Roteador Físico (pfSense WAN)
*   **Sintoma:** O pfSense configurado em modo Bridge (conectado à placa Wi-Fi física do notebook host) não conseguia obter um endereço IP público estável e perdia a conexão externa.
*   **Causa:** O roteador físico residencial bloqueava múltiplos endereços MACs de forma simultânea no mesmo canal Wi-Fi por motivos de segurança interna.
*   **Solução:** Alteração manual da interface WAN do pfSense (Adaptador 1 no VirtualBox) para o Modo NAT. Desta forma, o sistema operacional host passou a mascarar e traduzir todo o tráfego do pfSense sob o seu próprio endereço IP físico, garantindo acesso estável à internet.

### Desafio 2: Estouro de Armazenamento do Wazuh SIEM (VM 03)
*   **Sintoma:** A interface administrativa web do Wazuh Dashboard exibia o erro `Wazuh dashboard server is not ready yet` ou `Something went wrong`, e o banco de dados `wazuh-indexer` ficava travado indefinidamente no status de inicialização (`activating`).
*   **Análise Forense de Logs:** A análise do arquivo de eventos `/var/log/wazuh-indexer/wazuh-cluster.log` revelou a seguinte exceção: `java.io.IOException: No space left on device`. O disco virtual original de 25 GB havia sido totalmente consumido devido ao volume de logs de boot.
*   **Solução em 3 Fases:**
    1.  *Fase Física:* Desligamento seguro da VM e expansão do arquivo de disco virtual (`.vdi`) de 25 GB para 60 GB por meio do Gerenciador de Mídias do VirtualBox.
    2.  *Fase Lógica (Particionamento Linux):* Inicialização da VM e execução do comando `sudo growpart /dev/sda 2` (para estender a tabela de partição ativa) seguido de `sudo resize2fs /dev/sda2` (para redimensionar o sistema de arquivos ext4 online). O consumo de disco caiu de 100% para 44%, restabelecendo o espaço livre.
    3.  *Desbloqueio de Escrita no Elasticsearch/Opensearch:* O estouro de disco ativou a trava de segurança nativa do indexador que impede novas gravações (*read-only*). Para remover o bloqueio, enviamos uma chamada REST direta para a API do indexador (aplicada especificamente aos índices do Wazuh):
        ```bash
        curl -XPUT -k -u admin:[SENHA_MESTRE] "https://localhost:9200/wazuh-*/_settings" -H 'Content-Type: application/json' -d'{"index.blocks.read_only_allow_delete": null}'
        ```

### Desafio 3: Exclusão Acidental de Binários por Interrupção de Script
*   **Sintoma:** Ao executar a ferramenta de gerência de credenciais, o console retornava `line 67: /var/ossec/bin/wazuh-keystore: No such file or directory` e o serviço `wazuh-manager` falhava ao iniciar.
*   **Causa:** O script foi interrompido com um cancelamento forçado (`Ctrl + C`) no meio de uma reconfiguração com a flag `--overwrite`. O script realizou a exclusão preventiva dos binários do diretório `/var/ossec/bin/` mas foi cancelado antes de extrair os novos arquivos para o mesmo local.
*   **Solução:** Executada uma reinstalação completa e limpa do zero usando o comando do instalador com o parâmetro `--overwrite` completo. Com o disco agora saudável (60 GB), a instalação foi finalizada com sucesso e todos os binários foram integralmente restaurados.

### Desafio 4: Erro de Sincronia de Credenciais do Wazuh Dashboard
*   **Sintoma:** O login na interface web administrativa retornava constantemente `Invalid username or password` mesmo utilizando a senha mestre de administrador correta.
*   **Causa:** Devido aos travamentos anteriores por falta de espaço em disco, o arquivo de configuração do painel web (`/etc/wazuh-dashboard/wazuh.yml`) não gravou de forma síncrona a mesma senha de acesso cadastrada no banco de dados (indexer).
*   **Solução:** Utilização do utilitário oficial de redefinição de senhas do Wazuh para forçar a sincronização de credenciais de todos os módulos de forma unificada:
    ```bash
    sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh -u admin -p 'NovaSenhaSegura123!'
    ```

### Desafio 5: Erro de Sintaxe de Dicionário no FreeRADIUS
*   **Sintoma:** O comando `sudo systemctl restart freeradius` falhava com o status de erro `control process exited with error code`.
*   **Causa:** O arquivo `/etc/freeradius/3.0/users` continha o termo `Cleartexte-password` (com a letra "e" extra ao final de Cleartext e a letra "p" minúscula). O interpretador de dicionário do FreeRADIUS é extremamente rígido com a sintaxe e rejeitou a inicialização devido ao atributo desconhecido.
*   **Solução:** Correção da linha de texto para o padrão oficial exigido pelo dicionário do sistema: `Cleartext-Password` (com o "P" maiúsculo e sem a letra "e" sobressalente), reestabelecendo a inicialização imediata do serviço.

### Desafio 6: Conflito de Duplo Endereçamento IP na Estação do Usuário (VM 05)
*   **Sintoma:** O usuário realizava o login com sucesso no Captive Portal do pfSense, mas o navegador continuava sem carregar as páginas externas à rede.
*   **Causa:** A placa de rede `enp0s3` da VM do usuário possuía dois IPs ativos simultaneamente na mesma interface: o IP estático antigo `192.168.10.100` e o IP dinâmico `192.168.10.102` gerado pelo DHCP. O pfSense autorizava as requisições de saída do IP `.100` que enviou o formulário do portal, mas o sistema operacional da VM tentava encaminhar a navegação subsequente utilizando a rota do IP `.102` (que não estava autorizado), gerando o bloqueio preventivo pelo firewall.
*   **Solução:** Exclusão manual do perfil estático legado na interface gráfica de conexões de rede do Lubuntu e reinício completo da VM 05. A placa inicializou limpa com apenas o IP dinâmico ativo (`192.168.10.102`), sincronizando de forma perfeita a autenticação e o tráfego de saída do gateway.
