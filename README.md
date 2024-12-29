# **Configuração do Ambiente de Desenvolvimento**

## **Introdução**

Este guia fornece a configuração final e completa do ambiente de desenvolvimento local para o projeto do framework, integrando diversos componentes de DevOps para fornecer uma solução robusta e abrangente. O ambiente inclui:

- **Traefik** como load balancer e proxy reverso.
- **Grafana** para visualização de logs, métricas e traces.
- **Prometheus** para coleta de métricas.
- **Jaeger** para rastreamento distribuído.
- **OpenTelemetry Collector** para coleta e exportação de dados de telemetria.
- **Loki** e **Promtail** para agregação e coleta de logs.
- **Aplicação Node.js** com RxJS e instrumentação completa usando OpenTelemetry.

## **Objetivo**

Criar um ambiente de desenvolvimento local que permita:

- Monitorar e visualizar logs, métricas e traces da aplicação.
- Integrar ferramentas populares de observabilidade.
- Proporcionar um setup que possa ser facilmente reproduzido e ajustado conforme necessário.

## **Arquitetura e Design**

A arquitetura consiste em vários serviços Docker orquestrados pelo **Docker Compose**. O **Traefik** atua como proxy reverso, roteando o tráfego para os serviços apropriados. O **OpenTelemetry Collector** coleta dados de telemetria da aplicação e os exporta para o **Prometheus**, **Jaeger** e **Loki**. O **Grafana** é usado para visualizar métricas, logs e traces. A aplicação Node.js está instrumentada para enviar dados de telemetria para o **OpenTelemetry Collector**.

## **Componentes**

- **Aplicação Node.js**: Aplicação principal instrumentada com OpenTelemetry.
- **Traefik**: Proxy reverso e load balancer.
- **OpenTelemetry Collector**: Coleta e exporta dados de telemetria.
- **Prometheus**: Sistema de monitoramento e coleta de métricas.
- **Grafana**: Plataforma de visualização e análise.
- **Jaeger**: Sistema de rastreamento distribuído.
- **Loki**: Sistema de agregação de logs.
- **Promtail**: Coleta e envia logs para o Loki.

## **Estrutura de Pastas**

```
└── 📁devops
    └── 📁grafana
        └── 📁provisioning
            └── 📁dashboards
                └── dashboard.yml
            └── 📁datasources
                └── datasource.yml
    └── 📁loki
        └── config.yml
    └── 📁otel-collector
        └── otel-collector.yml
    └── 📁prometheus
        └── prometheus.yml
    └── 📁promtail
        └── config.yml
    └── 📁traefik
        └── acme.json
        └── dynamic_conf.yml
        └── traefik.yml
    └── compose.yaml
```

## **Configurações e Arquivos**

### **1. Arquivo `docker-compose.yml`**

Este arquivo define todos os serviços necessários para o ambiente.

```yaml
networks:
  shared:
    driver: bridge
    name: shared
    external: true

volumes:
  grafana_data:
  prometheus_data:

services:
  traefik:
    image: traefik:v2.4
    command:
      - --api.insecure=true
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --providers.file.directory=/etc/traefik/dynamic_conf
      - --providers.file.watch=true
    ports:
      - "80:80"
      - "8080:8080" # Dashboard do Traefik
    volumes:
      - ./traefik/traefik.yml:/etc/traefik/traefik.yml:ro
      - ./traefik/dynamic_conf.yml:/etc/traefik/dynamic_conf/dynamic_conf.yml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/acme.json:/acme.json
    networks:
      - shared
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`traefik.localhost`)"
      - "traefik.http.routers.traefik.service=api@internal"
      - "traefik.http.services.traefik.loadbalancer.server.port=8080"

  # app:
  #   build:
  #     context: ./app
  #     dockerfile: Dockerfile
  #   volumes:
  #     - ./app:/usr/src/app
  #   labels:
  #     - "traefik.enable=true"
  #     - "traefik.http.routers.app.rule=Host(`app.localhost`)"
  #     - "traefik.http.services.app.loadbalancer.server.port=3000"
  #   depends_on:
  #     - otel-collector
  #   networks:
  #     - traefik
  #   environment:
  #     - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
  #     - OTEL_TRACES_EXPORTER=otlp
  #     - OTEL_METRICS_EXPORTER=otlp
  #     - OTEL_LOGS_EXPORTER=otlp
  #     - OTEL_RESOURCE_ATTRIBUTES=service.name=app

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - prometheus_data:/prometheus
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    networks:
      - shared
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.prometheus.rule=Host(`prometheus.localhost`)"
      - "traefik.http.services.prometheus.loadbalancer.server.port=9090"

  jaeger:
    image: jaegertracing/all-in-one:latest
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
      COLLECTOR_OTLP_GRPC_PORT: "4317"
    ports:
      - "16686:16686" # UI do Jaeger
      - "4317:4317"   # Porta OTLP do Jaeger
    networks:
      - shared
    depends_on: []
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jaeger.rule=Host(`jaeger.localhost`)"
      - "traefik.http.services.jaeger.loadbalancer.server.port=16686"

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./otel-collector/otel-collector.yml:/etc/otel-collector-config.yml
    command: ["--config=/etc/otel-collector-config.yml"]
    ports:
      - "4318:4317"
      - "4319:4318"
      - "8889:8889"
    networks:
      - shared
    depends_on:
      - jaeger

  grafana:
    image: grafana/grafana:latest
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports:
      - "3000:3000"
    networks:
      - shared
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.localhost`)"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"

  loki:
    image: grafana/loki:2.2.1
    volumes:
      - ./loki/config.yml:/etc/loki/local-config.yaml
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - shared
    ports:
      - "3100:3100"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.loki.rule=Host(`loki.localhost`)"
      - "traefik.http.services.loki.loadbalancer.server.port=3100"

  promtail:
    image: grafana/promtail:2.2.1
    volumes:
      - ./promtail/config.yml:/etc/promtail/config.yml
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yml
    networks:
      - shared
    depends_on:
      - loki
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M
```

#### **1.1 Configurações da Docker Network**

A rede `shared` é usada para conectar todos os serviços. Ela é definida como uma rede externa para permitir a comunicação entre os contêineres.

```bash
docker network create shared --driver bridge
```

### **2. Arquivo `otel-collector/otel-collector.yml`**

Configuração do OpenTelemetry Collector com ajustes para integração de logs.

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4318
      http:
        endpoint: 0.0.0.0:4319

processors:
  batch:
    timeout: 1s
    send_batch_size: 1000

exporters:
  otlp:
    endpoint: "jaeger:4317"
    tls:
      insecure: true
    retry_on_failure:
      enabled: true
      initial_interval: 5s
      max_interval: 30s
      max_elapsed_time: 300s
  prometheus:
    endpoint: 0.0.0.0:8889
    namespace: "otel"
    const_labels:
      environment: "production"
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  telemetry:
    logs:
      level: "debug"
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
```

### **3. Arquivo `prometheus/prometheus.yml`**

Configuração do Prometheus para coletar métricas do OpenTelemetry Collector e da aplicação.

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'otel-collector'
    static_configs:
      - targets: ['otel-collector:8889']
    metrics_path: '/metrics'
    scheme: 'http'

  # - job_name: 'my-app'
  #   static_configs:
  #     - targets: ['app:9464'] # Se estiver expondo métricas diretamente da aplicação
```

### **4. Arquivos do Grafana**

#### **4.1 `grafana/provisioning/datasources/datasource.yml`**

Configuração das fontes de dados no Grafana.

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true

  - name: Jaeger
    type: jaeger
    access: proxy
    url: http://jaeger:16686

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
```

### **5. Arquivo `loki/config.yml`**

Configuração do Loki para agregação de logs.

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
    final_sleep: 0s
  chunk_idle_period: 5m
  chunk_retain_period: 30s
  max_transfer_retries: 0

schema_config:
  configs:
    - from: 2021-01-01
      store: boltdb
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 168h

storage_config:
  boltdb:
    directory: /tmp/loki/index

  filesystem:
    directory: /tmp/loki/chunks

limits_config:
  enforce_metric_name: false

chunk_store_config:
  max_look_back_period: 0s

table_manager:
  retention_deletes_enabled: true
  retention_period: 168h  # 7 dias
```

### **6. Arquivo `promtail/config.yml`**

Configuração do Promtail para coletar logs e enviá-los para o Loki.

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: containers
    static_configs:
      - targets:
          - localhost
        labels:
          job: containerlogs
          __path__: /var/lib/docker/containers/*/*log
          container_label: "logging=enabled"

    pipeline_stages:
      - json:
          expressions:
            stream: stream
            message: log
            timestamp: time
      - labels:
          stream
      - timestamp:
          source: timestamp
          format: RFC3339Nano
```

### **7. Configuração do Traefik**

#### **7.1 `traefik/traefik.yml`**

Configuração do Traefik.

```yaml
api:
  dashboard: true

log:
  level: DEBUG

entryPoints:
  web:
    address: ":80"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
  file:
    directory: "/etc/traefik/dynamic_conf"
    watch: true
```

#### **7.2 `traefik/dynamic_conf.yml`**

Este arquivo pode estar vazio ou ser usado para configurações dinâmicas adicionais.

```yaml
# Configurações dinâmicas adicionais podem ser adicionadas aqui
```

#### **7.3 `traefik/acme.json`**

Arquivo para armazenar certificados SSL. Deve ter permissão `600`.

```bash
touch traefik/acme.json
chmod 600 traefik/acme.json
```

### **8. Configuração da Aplicação**

#### **8.1 `app/package.json`**

Certifique-se de incluir as dependências necessárias para o OpenTelemetry e outros pacotes.

```json
{
  "name": "my-app",
  "version": "1.0.0",
  "description": "Aplicação Node.js com RxJS e OpenTelemetry",
  "main": "dist/index.js",
  "scripts": {
    "start": "node dist/index.js",
    "build": "tsc",
    "dev": "ts-node src/index.ts"
  },
  "dependencies": {
    "@opentelemetry/api": "^1.0.0",
    "@opentelemetry/sdk-node": "^1.0.0",
    "@opentelemetry/auto-instrumentations-node": "^0.27.0",
    "@opentelemetry/sdk-logs": "^0.27.0",
    "@opentelemetry/resources": "^1.0.0",
    "@opentelemetry/exporter-logs-otlp-http": "^0.27.0",
    "express": "^4.17.1",
    "rxjs": "^7.0.0"
  },
  "devDependencies": {
    "typescript": "^4.0.0",
    "ts-node": "^10.0.0"
  }
}
```

#### **8.2 `app/tsconfig.json`**

Configuração do TypeScript.

```json
{
  "compilerOptions": {
    "outDir": "./dist",
    "module": "commonjs",
    "target": "es2019",
    "strict": true,
    "esModuleInterop": true,
    "sourceMap": true
  },
  "include": ["src/**/*"]
}
```

#### **8.3 `app/src/index.ts`**

Implementação da aplicação com instrumentação completa.

```typescript
import express from 'express';
import { diag, DiagConsoleLogger, DiagLogLevel } from '@opentelemetry/api';
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-grpc';
import { OTLPLogExporter } from '@opentelemetry/exporter-logs-otlp-http';
import { ConsoleLogger } from '@opentelemetry/sdk-logs';

diag.setLogger(new DiagConsoleLogger(), DiagLogLevel.INFO);

const traceExporter = new OTLPTraceExporter({
  url: 'http://otel-collector:4317',
});

const logExporter = new OTLPLogExporter({
  url: 'http://otel-collector:4318/v1/logs',
});

const sdk = new NodeSDK({
  traceExporter: traceExporter,
  logExporter: logExporter,
  instrumentations: [getNodeAutoInstrumentations()],
  serviceName: 'app',
});

sdk.start()
  .then(() => {
    const app = express();
    const port = 3000;

    // Logger do OpenTelemetry
    const logger = new ConsoleLogger();

    app.get('/', (req, res) => {
      logger.info('Requisição recebida em /');
      res.send('Hello, OpenTelemetry with Logs!');
    });

    app.listen(port, () => {
      console.log(`App listening at http://localhost:${port}`);
    });
  })
  .catch((error) => {
    console.error('Erro inicializando o SDK OpenTelemetry', error);
  });
```

### **9. Arquivo `app/Dockerfile`**

Dockerfile para a aplicação.

```dockerfile
FROM node:14

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

## **Execução do Ambiente**

### **1. Construção da Aplicação**

No diretório `app`, execute:

```bash
npm run build
```

### **2. Iniciando o Ambiente com Docker Compose**

Na raiz do projeto, execute:

```bash
docker-compose up -d
```

### **3. Verificação dos Serviços**

- **Aplicação**: <http://app.localhost/>
- **Traefik Dashboard**: <http://traefik.localhost:8080/>
- **Prometheus**: <http://prometheus.localhost/>
- **Grafana**: <http://grafana.localhost/> (Usuário: `admin`, Senha: `admin`)
- **Jaeger UI**: <http://jaeger.localhost/>
- **Loki**: <http://loki.localhost/>

**Nota**: Certifique-se de adicionar entradas no arquivo `hosts` para mapear `*.localhost` para `127.0.0.1`.

No arquivo `/etc/hosts` (ou equivalente no Windows), adicione:

```
127.0.0.1 app.localhost
127.0.0.1 traefik.localhost
127.0.0.1 prometheus.localhost
127.0.0.1 grafana.localhost
127.0.0.1 jaeger.localhost
127.0.0.1 loki.localhost
```

## **Visualização no Grafana**

### **1. Acessando o Grafana**

- Acesse o Grafana em <http://grafana.localhost/>.
- Faça login com as credenciais padrão (usuário: `admin`, senha: `admin`).

### **2. Visualizando Métricas**

- Vá para **Dashboards** e importe dashboards pré-configurados ou crie os seus próprios.
- Utilize a fonte de dados **Prometheus** para visualizar métricas.

### **3. Visualizando Logs**

- Vá para **Explore** no menu lateral.
- Selecione a fonte de dados **Loki**.
- Use consultas como `{service="app"}` para visualizar os logs da aplicação.

### **4. Visualizando Traces**

- No menu **Explore**, selecione a fonte de dados **Jaeger**.
- Pesquise pelos traces da aplicação.

## **Conclusão**

Este setup fornece um ambiente de desenvolvimento completo com monitoramento e observabilidade integrados, incluindo logs, métricas e rastreamento distribuído.

### **Benefícios**

- **Observabilidade Completa**: Coleta e visualização de métricas, logs e traces em um único ambiente.
- **Flexibilidade**: Fácil de ajustar e estender conforme necessário.
- **Reprodutibilidade**: Configuração consistente que pode ser compartilhada entre membros da equipe.

### **Próximos Passos**

- **Personalizar Dashboards e Alertas**: Ajuste os dashboards no Grafana e configure alertas para monitoramento proativo.
- **Implementar Testes Automatizados**: Adicione testes para garantir a qualidade do código e do ambiente.
- **Explorar Mais Instrumentações**: Utilize mais recursos do OpenTelemetry para obter insights mais profundos.

## **Observações Finais**

- **Recursos do Sistema**: Certifique-se de que sua máquina possui recursos suficientes para executar todos os serviços Docker.
- **Segurança**: Para ambientes de produção, implemente medidas de segurança adequadas, como certificados SSL e autenticação robusta.
- **Atualizações**: Mantenha os componentes atualizados para aproveitar as últimas melhorias e correções de segurança.
