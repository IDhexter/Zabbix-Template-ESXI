# Templates Zabbix Customizados para VMware ESXi (via SNMP)

Este repositório contém templates customizados para monitoramento de hosts **VMware ESXi** no Zabbix através do protocolo SNMP. Os templates foram otimizados e ajustados para evitar alarmes falsos de rede e fornecer um monitoramento limpo de CPU, Memória e Interfaces de Rede.

---

## 📋 Templates Disponíveis

### 1. [Template v2.0 (Limpo)](./vmware_esxi_snmp_template.yaml)
*   **Versão:** 2.0
*   **Características:** Monitora inventário, CPU (média de cores), Memória Física (Real Memory) e descobre automaticamente as portas de rede.
*   **Diferencial:** **Não possui alertas/triggers de Link Down nas placas de rede.** Ideal para servidores que possuem portas físicas desconectadas por opção de projeto (portas sobressalentes), evitando alarmes falsos de cabos desconectados.

### 2. [Template v2.1 (Com Ethernet)](./vmware_esxi_snmp_template_v2.1_with_ethernet.yaml)
*   **Versão:** 2.1 (Com alertas de rede)
*   **Características:** Mesmas coletas do v2.0, mas inclui **triggers inteligentes de status de link de rede** (`Interface: Link down`).
*   **Comportamento do Alarme Inteligente:**
    *   Se a interface estiver habilitada no ESXi (**Administrativamente UP**), mas o cabo físico for desconectado (ou a porta do switch cair), o Zabbix **irá alarmar**.
    *   Se a interface for desativada propositalmente nas configurações do ESXi (**Administrativamente DOWN**), o Zabbix **não irá alarmar** (tratando a porta desligada como manutenção planejada e evitando alertas desnecessários).

### 3. [Template v3.0 (Completo - Sem Redes Físicas)](./vmware_esxi_snmp_template_v3.0_full.yaml)
*   **Versão:** 3.0 (Completo - Sem Placas de Rede)
*   **Características:** A versão mais robusta de monitoramento SNMP puro para o ESXi. Ela realiza a descoberta e monitoramento de:
    *   **CPU e Memória RAM:** Consumo médio, consumo por núcleo físico de CPU e memória RAM física (excluindo swap/memórias virtuais).
    *   **Datastores / Discos:** Descoberta automática de Datastores/Volumes de disco com alertas de capacidade (Atenção a 85% e Crítico a 95%).
    *   **Máquinas Virtuais (VMs):** Descoberta automática de todas as VMs rodando no host, coletando Estado de Energia (Ligada/Desligada/Suspensa), CPUs virtuais alocadas, RAM alocada e Sistema Operacional.
*   **Diferencial:**
    *   **Exclui totalmente a descoberta de placas de rede físicas (`vmnic`)** para evitar alarmes falsos de portas desconectadas.
    *   **Não inclui coleta de ICMP Ping**, evitando conflitos de chaves duplicadas (`icmpping`) caso você já use outro template dedicado para ping (ex: `PowerOFF`).
    *   **Habilita o fechamento manual (Manual Close)** em todos os alarmes de Datastore e Máquinas Virtuais.

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
