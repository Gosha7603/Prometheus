# -*- mode: ruby -*-
# vi: set ft=ruby -*-

Vagrant.configure("2") do |config|
  # Используем образ Ubuntu 22.04
  config.vm.box = "ubuntu/jammy64"
  config.vm.hostname = "monitoring-server"

  # Указываем провайдер VirtualBox для использования
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
  end

  # Сетевые настройки
  config.vm.network "private_network", type: "dhcp"
  config.vm.network "forwarded_port", guest: 9090, host: 9090  # Prometheus
  config.vm.network "forwarded_port", guest: 3000, host: 3000  # Grafana
  config.vm.network "forwarded_port", guest: 9100, host: 9100  # Node Exporter

  # Провизия (установка пакетов и настройка сервисов)
  config.vm.provision "shell", inline: <<-SHELL
    set -e

    # Обновление системы
    sudo apt-get update
    sudo apt-get upgrade -y

    # Установка Prometheus
    sudo useradd --no-create-home --shell /bin/false prometheus
    sudo mkdir -p /etc/prometheus /var/lib/prometheus
    sudo chown prometheus:prometheus /etc/prometheus /var/lib/prometheus

    wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
    tar xfz prometheus-2.45.0.linux-amd64.tar.gz
    sudo cp prometheus-2.45.0.linux-amd64/prometheus \
         prometheus-2.45.0.linux-amd64/promtool /usr/local/bin/
    sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool

    # Конфигурация Prometheus
    sudo tee /etc/prometheus/prometheus.yml > /dev/null <<EOF
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
EOF

    # Systemd сервис для Prometheus
    sudo tee /etc/systemd/system/prometheus.service > /dev/null <<EOF
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/bin/prometheus \\
    --config.file /etc/prometheus/prometheus.yml \\
    --storage.tsdb.path /var/lib/prometheus/ \\
    --web.console.templates=/etc/prometheus/consoles \\
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
EOF

    sudo systemctl daemon-reload
    sudo systemctl enable --now prometheus

    # Установка Grafana
    sudo apt-get install -y apt-transport-https software-properties-common
    wget -q -O /tmp/grafana.gpg https://packages.grafana.com/gpg.key
    sudo gpg --dearmor -o /usr/share/keyrings/grafana.gpg /tmp/grafana.gpg
    echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
    sudo apt-get update
    sudo apt-get install -y grafana

    sudo systemctl enable --now grafana-server

    # Установка Node Exporter
    sudo useradd --no-create-home --shell /bin/false node_exporter
    wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
    tar xfz node_exporter-1.6.1.linux-amd64.tar.gz
    sudo cp node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
    sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter

    # Systemd сервис для Node Exporter
    sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

    sudo systemctl daemon-reload
    sudo systemctl enable --now node_exporter

    # Очистка
    rm -rf prometheus-* node_exporter-*

    # Проверка статусов
    echo "Проверка сервисов:"
    systemctl status prometheus grafana-server node_exporter --no-pager || true
  SHELL
end