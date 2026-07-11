# 3.2. Infraestrutura Local e Serviços de Mensageria (Docker Compose)

Esta etapa documenta o provisionamento do ambiente conteinerizado responsável por receber o payload JSON via protocolo MQTT vindo da borda (Edge/Kepware) e disponibilizá-lo para a ingestão escalável no ambiente de nuvem e posterior processamento Lakehouse.

## 3.2.1. Arquitetura de Containers (Docker Compose)

O arquivo `docker-compose.yml` centraliza os serviços locais de mensageria e persistência temporária de telemetria. Ele garante isolamento de rede através de uma subnet dedicada industrial (`ot_it_bridge_net`).

```yaml
version: '3.8'

services:
  # Broker MQTT de Alta Performance para recepção das tags industriais
  mosquitto_broker:
    image: eclipse-mosquitto:2.0.15
    container_name: mqtt_industrial_broker
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
    ports:
      - "1883:1883"   # Porta nativa sem TLS (ambiente de teste/desenvolvimento)
      - "8883:8883"   # Porta segura com TLS para produção
    networks:
      - ot_it_bridge_net
    restart: always

  # Serviço de persistência de séries temporais para redundância local de auditoria
  influxdb_local:
    image: influxdb:2.7-alpine
    container_name: influx_telemetria_local
    ports:
      - "8086:8086"
    volumes:
      - influxdb_data:/var/lib/influxdb2
    networks:
      - ot_it_bridge_net
    restart: on-failure

networks:
  ot_it_bridge_net:
    driver: bridge

volumes:
  influxdb_data:
    driver: local


# Inicialização da stack de mensageria
docker-compose up -d

# Verificação do status operacional dos containers
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

NAMES                     STATUS              PORTS
mqtt_industrial_broker    Up 2 hours          0.0.0.0:1883->1883/tcp, 0.0.0.0:8883->8883/tcp
influx_telemetria_local   Up 2 hours          0.0.0.0:8086->8086/tcp
