Como Engenheiro DevOps e Arquiteto de Cloud, meu foco para este projeto é a implantabilidade contínua (CD) e a observabilidade proativa. Em uma arquitetura orientada a eventos, o maior desafio não é o deploy em si, mas garantir que a cadeia de eventos não se quebre e que tenhamos visibilidade total de gargalos nas filas ou falhas nos consumidores.

Abaixo, detalho a esteira de CI/CD e a estratégia de monitoramento.

---

## 1. Especificação do Fluxo CI/CD (GitHub Actions)

Utilizaremos uma estratégia de GitFlow com Deploy Baseado em Trunk para produção, garantindo que o código em main esteja sempre pronto para deploy via Blue/Green ou Canary para evitar downtime.

Workflow: `.github/workflows/pipeline-prod.yml`

```yaml
name: CI/CD Production Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  # --- ETAPA 1: INTEGRAÇÃO CONTÍNUA (CI) ---
  ci-stage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Environment
        uses: actions/setup-node@v3 # Exemplo para Node.js (Serviço de Webhook)
        with:
          node-version: '18'

      - name: Security Scan (SAST)
        uses: bridgecrewio/checkov-action@master # Verifica segurança no IaC (Terraform)
        
      - name: Unit & Integration Tests
        run: |
          npm install
          npm test

      - name: Build & Push Docker Image
        if: github.event_name == 'push'
        run: |
          docker build -t my-app:${{ github.sha }} .
          aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${{ secrets.ECR_REPO }}
          docker tag my-app:${{ github.sha }} ${{ secrets.ECR_REPO }}:latest
          docker push ${{ secrets.ECR_REPO }}:latest

  # --- ETAPA 2: ENTREGA CONTÍNUA (CD) ---
  cd-stage:
    needs: ci-stage
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to ECS (Blue/Green)
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: task-def.json
          service: webhook-service
          cluster: production-cluster
          wait-for-service-stability: true
          # Usamos o CodeDeploy aqui para gerenciar o tráfego Blue/Green
```

Diferenciais DevOps aplicados:

Imutabilidade: Cada build gera uma tag única baseada no SHA do Git.

Segurança: Varredura de vulnerabilidades na imagem Docker (Trivy ou Snyk) antes do push.

Zero Downtime: Integração com AWS CodeDeploy para realizar a troca de tráfego de forma gradual.

---

## 2. Gatilhos de Monitoramento e Alertas (AWS CloudWatch)

Em um ambiente EDA (Event-Driven), métricas de CPU/Memória são insuficientes. Precisamos monitorar a saúde do fluxo de eventos.

### A. Métricas Críticas e Gatilhos de Alerta

| Componente | Métrica CloudWatch | Gatilho (Alarm) | Ação Proativa |
| :--- | :--- | :--- | :--- |
| Webhook Inbound | 4XXError & 5XXError | > 1% das requisições em 5 min | Alerta no Slack/PagerDuty |
| SQS (Queue) | ApproximateAgeOfOldestMessage | > 60 segundos | Auto-scaling: Aumenta número de instâncias/consumers |
| SQS (DLQ) | NumberOfMessagesSent | > 0 mensagens | Investigação imediata (Falha de processamento) |
| WebSockets | NewConnectionCount | Queda súbita > 50% | Verificar saúde do NLB (Network Load Balancer) |
| Lambda (Consumers) | ConcurrentExecutions | > 80% do limite da conta | Solicitar aumento de quota ou throttling preventivo |

### B. Dashboard de Observabilidade (Logs & Tracing)

Não basta saber que falhou, precisamos saber onde na cadeia o evento parou:

CloudWatch Logs Insights:

Query para identificar latência no processamento de webhooks:

```
filter @message like /Processing completed/
| stats avg(processingTime) by bin(1m)
```

AWS X-Ray (Distributed Tracing):

Essencial para visualizar o caminho do evento: API Gateway -> SQS -> Lambda -> WebSocket.

Permite identificar qual microserviço está causando gargalo no fluxo assíncrono.

CloudWatch Synthetics (Canaries):

Um script que "finge" ser um sistema externo enviando um Webhook a cada 1 minuto. Se o sistema não processar e o WebSocket não receber a confirmação, o alarme dispara mesmo sem tráfego real.

---

## 3. Estratégia de Escalabilidade (Auto-Scaling)

Como Engenheiro DevOps, configuro o escalonamento baseado no comportamento da fila, não apenas no consumo de hardware:

Scale-out: Se ApproximateNumberOfMessagesVisible (mensagens esperando) subir, o CloudWatch dispara um alarme que escala o cluster ECS ou aumenta a concorrência das Lambdas.

Custo Otimizado: Durante a madrugada, o número mínimo de instâncias é reduzido, e utilizamos Fargate Spot para processamentos em background que não são sensíveis a tempo, reduzindo custos em até 70%.

Este desenho garante que o sistema seja resiliente a picos de carga (ex: Black Friday) e que a equipe de engenharia seja avisada de uma falha antes mesmo que o cliente perceba.
