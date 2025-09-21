Resumo

Projeto em Python para monitoramento contínuo de recursos de sistemas (servidores, VMs, workstations). Usa a biblioteca psutil para coletar métricas de CPU, memória, disco e rede; processa essas métricas em tempo real e dispara alertas (Slack, Microsoft Teams ou e-mail) quando os valores ultrapassam limites configuráveis. Pensado para ser leve, configurável por arquivo (YAML/JSON) e fácil de integrar em operações existentes.

Objetivos

Detectar proativamente degradações e picos de uso de recursos.

Notificar times responsáveis por canais já utilizados (Slack/Teams/email).

Permitir configuração flexível de thresholds, janelas de avaliação e escalonamento.

Gerar histórico mínimo para análise e correlação (logs / banco leve ou arquivos).

Ser simples de empacotar (Docker, systemd) e implementar em ambientes heterogêneos.

Funcionalidades principais

Coleta periódica de métricas:

CPU: uso total, por-core, load average (quando aplicável).

Memória: uso total, disponível, uso de swap.

Disco: uso por mount point, I/O (read/write bytes, ops).

Rede: tráfego por interface (bytes/s), erros, conexões ativas.

Regras de alerta configuráveis:

Thresholds por métrica (ex.: CPU > 85% por 2 minutos).

Janelas temporais e número de ocorrências antes de alertar.

Escalonamento (alerta inicial → lembrete → alerta crítico).

Múltiplos canais de notificação:

Slack (webhook ou bot).

Microsoft Teams (incoming webhook).

E-mail (SMTP com TLS).

Logging estruturado (JSON opcional) e relatórios periódicos.

Configuração via arquivo (YAML/JSON) e variáveis de ambiente.

Modo “dry-run” para testar regras sem enviar notificações reais.

Endpoints REST simples (opcional) para status/healthcheck.

Arquitetura e componentes

Collector (coletor) — módulo que usa psutil para ler métricas em intervalos (ex.: 5s).

Processor / Aggregator — aplica janelas (tumbling/sliding), calcula médias/picos e decide sobre thresholds.

Alert Manager — lógica de debounce/escalation e persistência temporária de ocorrências (em memória ou Redis).

Notifier — adaptadores para Slack, Teams, SMTP.

Storage (opcional) — SQLite/InfluxDB/arquivos para manter histórico curto.

API / UI (opcional) — endpoint para checar status, silenciar alertas ou ver métricas resumidas.

Config & CLI — carregar arquivo de configuração, parâmetros de runtime.

Fluxo básico: Collector → Processor → Alert Manager → Notifier → (logs / storage).

Exemplo de métricas e thresholds sugeridos

CPU total: alerta se > 85% por 2 minutos consecutivos.

CPU por core: alerta se qualquer core > 95% por 1 minuto.

Memória usada: alerta se uso > 90% ou memória disponível < 500MB.

Swap: alerta se swap > 30% ou swaps/sec alto contínuo.

Disco (mount /): alerta se uso > 90% ou I/O wait alto (> 20%).

Rede (eth0): alerta se tráfego out/in > X Mbps (definir por perfil).

Número de conexões TCP ESTABLISHED > N (indicador de DoS ou leak).
