# monitoramento_recursos_sistema.py
"""
Monitoramento de recursos do sistema
-----------------------------------
- Coleta métricas de CPU, memória, disco e rede usando psutil.
- Envia alertas via Slack, Microsoft Teams (webhook) ou e-mail quando métricas ultrapassam limites configuráveis.
- Configurável via arquivo YAML (thresholds, canais de alerta, intervalo, cooldown etc.).
- Tem modo dry-run, logging e proteção contra alertas repetidos (cooldown).

Requisitos:
    pip install psutil pyyaml requests

Uso:
    sudo python3 monitoramento_recursos_sistema.py --config config.yml

Observações:
- Requer permissões adequadas para coletar métricas de sistema (normalmente não precisa de root).
- Para envio de e-mail TLS, informe servidor SMTP, porta, usuário e senha no config (use vault/env vars em produção).
"""

import argparse
import time
import psutil
import yaml
import logging
import requests
import smtplib
import socket
from email.message import EmailMessage
from typing import Dict, Any


DEFAULT_CONFIG = {
    'interval_seconds': 30,
    'cooldown_seconds': 300,
    'thresholds': {
        'cpu_percent': 85,
        'memory_percent': 85,
        'disk_percent': 90,
        'net_sent_bytes_per_sec': 10485760,  # 10 MB/s
        'net_recv_bytes_per_sec': 10485760,
    },
    'alerts': {
        'slack_webhook': None,
        'teams_webhook': None,
        'email': None,  # dict with smtp_server, smtp_port, username, password, from, to (list)
    },
}


class AlertRateLimiter:
    """Evita envio repetido de alertas durante um período de cooldown."""

    def __init__(self, cooldown_seconds: int):
        self.cooldown = cooldown_seconds
        self.last_alert_time: Dict[str, float] = {}

    def should_alert(self, key: str) -> bool:
        now = time.time()
        last = self.last_alert_time.get(key)
        if last is None or (now - last) >= self.cooldown:
            self.last_alert_time[key] = now
            return True
        return False


def load_config(path: str) -> Dict[str, Any]:
    cfg = DEFAULT_CONFIG.copy()
    try:
        with open(path, 'r', encoding='utf-8') as f:
            user_cfg = yaml.safe_load(f) or {}
            # deep update
            def deep_update(d, u):
                for k, v in u.items():
                    if isinstance(v, dict) and isinstance(d.get(k), dict):
                        deep_update(d[k], v)
                    else:
                        d[k] = v
            deep_update(cfg, user_cfg)
    except FileNotFoundError:
        logging.warning(f"Config file {path} not found. Using defaults.")
    return cfg


def gather_metrics(prev_net: Dict[str, int], interval: int):
    cpu = psutil.cpu_percent(interval=None)
    mem = psutil.virtual_memory().percent
    disk = psutil.disk_usage('/').percent

    net_io = psutil.net_io_counters()
    net = {
        'bytes_sent': net_io.bytes_sent,
        'bytes_recv': net_io.bytes_recv,
    }
    net_sent_rate = None
    net_recv_rate = None
    if prev_net:
        dt = interval
        net_sent_rate = (net['bytes_sent'] - prev_net['bytes_sent']) / dt
        net_recv_rate = (net['bytes_recv'] - prev_net['bytes_recv']) / dt

    metrics = {
        'cpu_percent': cpu,
        'memory_percent': mem,
        'disk_percent': disk,
        'net_sent_bps': net_sent_rate,
        'net_recv_bps': net_recv_rate,
    }
    return metrics, net


# --------- Alert senders ---------

def send_slack(webhook: str, title: str, text: str, dry_run: bool = False):
    payload = {"text": f"*{title}*\n{text}"}
    if dry_run:
        logging.info(f"DRY RUN Slack payload: {payload}")
        return
    resp = requests.post(webhook, json=payload, timeout=10)
    resp.raise_for_status()


def send_teams(webhook: str, title: str, text: str, dry_run: bool = False):
    # Teams expects a simple JSON card
    payload = {
        "@type": "MessageCard",
        "@context": "http://schema.org/extensions",
        "summary": title,
        "themeColor": "FF0000",
        "title": title,
        "text": text,
    }
    if dry_run:
        logging.info(f"DRY RUN Teams payload: {payload}")
        return
    resp = requests.post(webhook, json=payload, timeout=10)
    resp.raise_for_status()


def send_email(email_cfg: Dict[str, Any], subject: str, body: str, dry_run: bool = False):
    msg = EmailMessage()
    msg['Subject'] = subject
    msg['From'] = email_cfg['from']
    msg['To'] = ', '.join(email_cfg['to']) if isinstance(email_cfg['to'], (list, tuple)) else email_cfg['to']
    msg.set_content(body)

    if dry_run:
        logging.info(f"DRY RUN Email to {msg['To']}: {subject}\n{body}")
        return

    server = smtplib.SMTP(email_cfg['smtp_server'], email_cfg.get('smtp_port', 587), timeout=10)
    try:
        server.ehlo()
        if email_cfg.get('use_tls', True):
            server.starttls()
            server.ehlo()
        username = email_cfg.get('username')
        password = email_cfg.get('password')
        if username and password:
            server.login(username, password)
        server.send_message(msg)
    finally:
        server.quit()


def format_metrics(metrics: Dict[str, Any]) -> str:
    lines = []
    for k, v in metrics.items():
        if v is None:
            lines.append(f"{k}: n/a")
        elif isinstance(v, float):
            if 'bps' in k:
                lines.append(f"{k}: {v:.2f} bytes/s")
            else:
                lines.append(f"{k}: {v:.1f}%")
        else:
            lines.append(f"{k}: {v}")
    return "\n".join(lines)


def build_alert_message(hostname: str, metric_name: str, value: Any, threshold: Any, metrics: Dict[str, Any]) -> (str, str):
    title = f"Alerta: {metric_name} alto em {hostname}"
    text = f"Métrica: {metric_name}\nValor: {value}\nLimite: {threshold}\n\nMétricas atuais:\n" + format_metrics(metrics)
    return title, text


def main():
    parser = argparse.ArgumentParser(description='Monitor de recursos do sistema com alertas')
    parser.add_argument('--config', '-c', default='config.yml', help='Arquivo de configuração YAML')
    parser.add_argument('--interval', '-i', type=int, help='Intervalo de coleta em segundos (sobrepõe config)')
    parser.add_argument('--dry-run', action='store_true', help='Não envia alertas, só mostra o que faria')
    parser.add_argument('--verbose', '-v', action='store_true', help='Ativa logging detalhado')

    args = parser.parse_args()

    logging.basicConfig(level=logging.DEBUG if args.verbose else logging.INFO, format='%(asctime)s %(levelname)s: %(message)s')

    cfg = load_config(args.config)
    interval = args.interval or cfg.get('interval_seconds', 30)
    cooldown = cfg.get('cooldown_seconds', 300)
    thresholds = cfg.get('thresholds', {})
    alerts_cfg = cfg.get('alerts', {})

    limiter = AlertRateLimiter(cooldown)
    prev_net = {}
    hostname = socket.gethostname()

    logging.info(f"Starting monitor: interval={interval}s, cooldown={cooldown}s")

    while True:
        try:
            metrics, prev_net = gather_metrics(prev_net, interval)

            # check each threshold
            # CPU
            cpu_thresh = thresholds.get('cpu_percent')
            if cpu_thresh is not None and metrics['cpu_percent'] >= cpu_thresh:
                key = f"cpu:{hostname}"
                if limiter.should_alert(key):
                    title, text = build_alert_message(hostname, 'CPU %', metrics['cpu_percent'], cpu_thresh, metrics)
                    if alerts_cfg.get('slack_webhook'):
                        send_slack(alerts_cfg['slack_webhook'], title, text, args.dry_run)
                    if alerts_cfg.get('teams_webhook'):
                        send_teams(alerts_cfg['teams_webhook'], title, text, args.dry_run)
                    if alerts_cfg.get('email'):
                        send_email(alerts_cfg['email'], title, text, args.dry_run)

            # Memory
            mem_thresh = thresholds.get('memory_percent')
            if mem_thresh is not None and metrics['memory_percent'] >= mem_thresh:
                key = f"memory:{hostname}"
                if limiter.should_alert(key):
                    title, text = build_alert_message(hostname, 'Memória %', metrics['memory_percent'], mem_thresh, metrics)
                    if alerts_cfg.get('slack_webhook'):
                        send_slack(alerts_cfg['slack_webhook'], title, text, args.dry_run)
                    if alerts_cfg.get('teams_webhook'):
                        send_teams(alerts_cfg['teams_webhook'], title, text, args.dry_run)
                    if alerts_cfg.get('email'):
                        send_email(alerts_cfg['email'], title, text, args.dry_run)

            # Disk
            disk_thresh = thresholds.get('disk_percent')
            if disk_thresh is not None and metrics['disk_percent'] >= disk_thresh:
                key = f"disk:{hostname}"
                if limiter.should_alert(key):
                    title, text = build_alert_message(hostname, 'Disco %', metrics['disk_percent'], disk_thresh, metrics)
                    if alerts_cfg.get('slack_webhook'):
                        send_slack(alerts_cfg['slack_webhook'], title, text, args.dry_run)
                    if alerts_cfg.get('teams_webhook'):
                        send_teams(alerts_cfg['teams_webhook'], title, text, args.dry_run)
                    if alerts_cfg.get('email'):
                        send_email(alerts_cfg['email'], title, text, args.dry_run)

            # Network sent/recv
            sent_thresh = thresholds.get('net_sent_bytes_per_sec')
            if sent_thresh is not None and metrics['net_sent_bps'] is not None and metrics['net_sent_bps'] >= sent_thresh:
                key = f"net_sent:{hostname}"
                if limiter.should_alert(key):
                    title, text = build_alert_message(hostname, 'Net sent B/s', metrics['net_sent_bps'], sent_thresh, metrics)
                    if alerts_cfg.get('slack_webhook'):
                        send_slack(alerts_cfg['slack_webhook'], title, text, args.dry_run)
                    if alerts_cfg.get('teams_webhook'):
                        send_teams(alerts_cfg['teams_webhook'], title, text, args.dry_run)
                    if alerts_cfg.get('email'):
                        send_email(alerts_cfg['email'], title, text, args.dry_run)

            recv_thresh = thresholds.get('net_recv_bytes_per_sec')
            if recv_thresh is not None and metrics['net_recv_bps'] is not None and metrics['net_recv_bps'] >= recv_thresh:
                key = f"net_recv:{hostname}"
                if limiter.should_alert(key):
                    title, text = build_alert_message(hostname, 'Net recv B/s', metrics['net_recv_bps'], recv_thresh, metrics)
                    if alerts_cfg.get('slack_webhook'):
                        send_slack(alerts_cfg['slack_webhook'], title, text, args.dry_run)
                    if alerts_cfg.get('teams_webhook'):
                        send_teams(alerts_cfg['teams_webhook'], title, text, args.dry_run)
                    if alerts_cfg.get('email'):
                        send_email(alerts_cfg['email'], title, text, args.dry_run)

            # Optional: log metrics
            logging.info(format_metrics(metrics))

            break

        except requests.RequestException as e:
            logging.error(f"Erro ao enviar alerta via webhook: {e}")
        except smtplib.SMTPException as e:
            logging.error(f"Erro ao enviar email: {e}")
        except KeyboardInterrupt:
            logging.info("Encerrando por KeyboardInterrupt")
            break
        except Exception as e:
            logging.exception(f"Erro inesperado: {e}")

        time.sleep(interval)


if __name__ == '__main__':
    main()



# -----------------------------
# Arquivo de exemplo: config.yml
# -----------------------------
# interval_seconds: 30
# cooldown_seconds: 300
# thresholds:
#   cpu_percent: 85
#   memory_percent: 85
#   disk_percent: 90
#   net_sent_bytes_per_sec: 10485760
#   net_recv_bytes_per_sec: 10485760
# alerts:
#   slack_webhook: "https://hooks.slack.com/services/XX/YYY/ZZZ"
#   teams_webhook: "https://outlook.office.com/webhook/..."
#   email:
#     smtp_server: "smtp.example.com"
#     smtp_port: 587
#     username: "alert@example.com"
#     password: "supersecret"
#     from: "alert@example.com"
#     to:
#       - "ops@example.com"
#     use_tls: true

# -----------------------------
# systemd unit example (monitor.service)
# -----------------------------
# [Unit]
# Description=Monitor de recursos do sistema
# After=network.target
#
# [Service]
# Type=simple
# ExecStart=/usr/bin/python3 /opt/monitoramento_recursos_sistema.py --config /etc/monitor/config.yml
# Restart=always
#
# [Install]
# WantedBy=multi-user.target

# -----------------------------
# requirements.txt
# -----------------------------
# psutil
# pyyaml
# requests

# README resumido
# -----------------------------
# 1) Instale dependências: pip install -r requirements.txt
# 2) Copie o script para /opt/ ou onde preferir e crie o config.yml em /etc/monitor
# 3) Teste em dry-run:
#    python3 monitoramento_recursos_sistema.py --config config.yml --dry-run --verbose
# 4) Configure systemd com o arquivo de exemplo e habilite o serviço.
# 5) Em produção, proteja credenciais (use Vault/variáveis de ambiente) e valide a política de envio de e-mails.
