# Templates Zabbix Customizados para VMware ESXi (via SNMP)

Este repositório contém templates customizados para monitoramento de hosts **VMware ESXi** no Zabbix através do protocolo SNMP. Os templates foram otimizados e ajustados para evitar alarmes falsos de rede e fornecer um monitoramento limpo de CPU, Memória e Interfaces de Rede.

---

## 📋 Templates Disponíveis - Detalhamento Completo

Neste repositório você encontra três opções de templates para atender diferentes cenários de infraestrutura:

### 1. [Template v2.0 (Limpo - Sem Alertas de Link de Rede)](./vmware_esxi_snmp_template.yaml)
*   **Versão:** 2.0 (Foco em Processamento e Memória)
*   **Indicação de Uso:** Ideal para servidores que possuem placas de rede desconectadas por opção de projeto (portas sobressalentes/reservas). Ele evita alertas falsos de "cabo desconectado" em portas não utilizadas.
*   **O que ele monitora em detalhes:**
    *   **Identificação do Sistema:** Nome do host (`system.name`) e descrição/versão do ESXi (`system.descr`).
    *   **Uptime:** Tempo online do host (`system.uptime`) com alerta de reboot caso o uptime seja menor que 10 minutos.
    *   **Processamento (CPU):** Uso médio geral de CPU (`vm.cpu.util`) com alarmes de Atenção (80%), Problema (90%) e Crítico (95%).
    *   **Cores de CPU (LLD):** Descobre e monitora dinamicamente a carga de cada núcleo individual de processamento.
    *   **Memória RAM Física (LLD):** Identifica a memória física instalada, monitorando o espaço total (B), usado (B) e percentual de uso (%), com alertas em 3 níveis (80%, 90% e 95%).
    *   **Placas de Rede (LLD):** Descobre as placas físicas de rede e monitora o status delas, **mas não cria nenhum alarme/trigger de queda de sinal (Link Down).**

---

### 2. [Template v2.1 (Com Ethernet Inteligente)](./vmware_esxi_snmp_template_v2.1_with_ethernet.yaml)
*   **Versão:** 2.1 (Foco em Processamento, Memória e Redes Físicas)
*   **Indicação de Uso:** Recomendado para ambientes onde o status físico de conexão dos cabos de rede é crítico, mas você deseja controle inteligente sobre portas desativadas.
*   **O que ele monitora em detalhes:**
    *   **Tudo do Template v2.0:** CPU média, núcleos individuais, RAM e uptime.
    *   **Alertas de Rede Inteligentes (Link Down):** Possui protótipo de gatilho para queda de link (`Interface {#IFNAME}: Link down`) configurado com lógica de dupla validação:
        *   **Dispara Alerta:** Se a placa operacional estiver desligada (`Operational Status = DOWN`), mas a placa estiver ativa nas configurações do ESXi (`Administrative Status = UP`).
        *   **Não Dispara Alerta:** Se a placa for desativada administrativamente no painel do ESXi (`Administrative Status = DOWN`).
        *   *Todos os alertas de queda de rede possuem suporte a fechamento manual (Manual Close) ativo.*

---

### 3. [Template v3.0 (Completo - Armazenamento e VMs - Sem Redes Físicas)](./vmware_esxi_snmp_template_v3.0_full.yaml)
*   **Versão:** 3.0 (Foco em Armazenamento, VMs e SNMP Puro)
*   **Indicação de Uso:** O mais completo para monitoramento profundo de servidores de virtualização autônomos. Ele ignora as placas de rede para evitar ruído de alarmes e não inclui ping ICMP para evitar conflitos com outros templates de conectividade (como o `PowerOFF`).
*   **O que ele monitora em detalhes:**
    *   **CPU e RAM:** Uso de CPU geral (médio), cores de CPU individuais e uso detalhado de memória RAM física (excluindo swap/memória virtual).
    *   **Datastores / Armazenamento (LLD):** Mapeia automaticamente todas as partições de disco e Datastores VMFS (onde ficam as máquinas virtuais). Coleta Espaço Total (GB), Espaço Usado (GB) e Porcentagem de uso, gerando alarmes de **Atenção (85%)** e **Crítico (95%)** com suporte a fechamento manual.
    *   **Máquinas Virtuais Internas (LLD):** Consulta a tabela do hypervisor (`vmwVmTable`) e descobre todas as VMs hospedadas no host, monitorando:
        *   Nome visível de cada VM.
        *   Estado de Energia (Ligada, Desligada, Suspensa).
        *   Quantidade de vCPUs e Memória RAM configurada para cada VM.
        *   Sistema Operacional instalado na VM.
        *   **Alerta de VM Desligada:** Avisa se uma máquina virtual for desligada ou suspensa (com suporte a fechamento manual).

### 4. [Template Network WAN Circuits Custom Thresholds](./zabbix_wan_circuits_thresholds_template.yaml)
*   **Versão:** 1.0 (Template para Roteadores/Links WAN)
*   **Indicação de Uso:** Automatiza 100% a criação de alertas e gráficos de tráfego nos 4 hosts de circuito (`IPN-RTR-NY01`, `IPN-RTR-NY02`, `RTR-NYEXT1`, `RTR-NYEXT3-XO`).
*   **O que ele faz automaticamente:**
    *   **Descoberta Inteligente (LLD):** Descobre apenas as interfaces especificadas (`Te0/1/2` e `Gi0/0/0`) usando um filtro regex.
    *   **Triggers Dinâmicas sem Configuração Manual:** Cria as 4 triggers separadas (IN 65%, IN 95%, OUT 65%, OUT 95%) calculando os limites **automaticamente** com base na velocidade real negociada da placa (ex: 6.5G/9.5G para placas de 10Gbps e 650M/950M para placas de 1Gbps).
    *   **Gráfico Automático com Escala Dinâmica:** Gera um gráfico de tráfego com altura de 300px, exibição de triggers ativa e **escala Y máxima travada na velocidade real da placa** (usando a métrica de velocidade como limite).

---

## 🧠 Entendimento Crítico: Métricas de CPU (Zabbix vs. vCenter)

Ao monitorar servidores ESXi robustos (como processadores com alto número de núcleos, ex: Intel Xeon Platinum 8562Y+ com **32 cores físicos** e **64 threads lógicas**), é comum observar uma discrepância entre o uso de CPU reportado pelo vCenter (ex: **80%**) e pelo Zabbix/SNMP (ex: **40%**). 

Ises números **não são erros de cálculo**, mas representam metodologias e denominadores diferentes:

| Característica / Métrica | SNMP (`hrProcessorLoad` / Zabbix) | vCenter (vSphere Client) |
| :--- | :--- | :--- |
| **Valor Reportado (Exemplo)**| **~40% (CPU Utilization)** | **~80% (CPU Usage)** |
| **Foco do Cálculo** | **Lógico** (Baseado em tempo/threads) | **Físico** (Baseado em recursos/cores) |
| **Denominador da Capacidade** | **64 Threads** (Processadores lógicos) | **32 Cores** (Processadores físicos reais) |
| **Frequência Dinâmica (Turbo)**| Não considera (trata o clock de forma estática) | Considera (pode elevar a métrica acima de 100%) |
| **Objetivo Principal** | Avaliar filas de agendamento de tarefas no SO | Avaliar a exaustão física real do hardware de silício |

### Qual é o correto reportar?
*   **Para relatórios de capacidade e investimentos (Diretoria/Gestão):** O correto é focar na métrica **Física (80% / vCenter)**, pois os 32 cores físicos reais representam o limite absoluto de hardware do equipamento (o Hyper-Threading não duplica o poder de processamento da CPU, ele apenas otimiza o fluxo de execução em ~20% a 35%).
*   **Para análise de concorrência de processos (Sysadmins):** A métrica lógica de 40% é útil para saber se as filas de tarefas do sistema operacional estão sobrecarregadas de threads.

### Como igualar os valores no Zabbix para bater com o vCenter?
1.  **Ajuste via SNMP:** Multiplique a fórmula do item calculado no Zabbix por **2** (visto que a proporção de Threads por Core Físico no hardware citado é de 2:1):
    ```text
    avg(last_foreach(//hrProcessorLoad[*])) * 2
    ```
2.  **Ajuste migrando para a API nativa da VMware:** Configure o Zabbix para usar a coleta direta do SDK da VMware e utilize o contador nativo `cpu/usage[average]` com o multiplicador de `0.01` em seu pré-processamento.

---

## 🛠️ Requisitos de Comunicação e Segurança

Dependendo do protocolo escolhido para monitoramento do ESXi, o firewall de rede e as credenciais devem ser configurados:

*   **Via SNMP (Lógico):**
    *   **Porta:** `161 UDP` liberada do Zabbix Server/Proxy para o host ESXi.
    *   **Configuração:** Macro `{$SNMP_COMMUNITY}` configurada com a community string correta.
*   **Via API/SDK da VMware (Físico):**
    *   **Porta:** `443 TCP` (HTTPS padrão) liberada do Zabbix Server/Proxy para o host ESXi ou vCenter.
    *   **Configuração do Zabbix Server:** Parâmetro `StartVMwareCollectors` configurado com valor maior que 0 no `/etc/zabbix/zabbix_server.conf`.
    *   **Credenciais de Acesso:** Usuário VMware com perfil **Read-Only (Somente Leitura)** e opção **"Propagate to children"** ativa (Garante segurança total, sem permissão de alteração do ambiente virtualizado).
