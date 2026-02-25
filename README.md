# OaaC — Observability as Code

**Catálogos YAML como fonte única da verdade para métricas, alertas, roteamento de alertas e dashboards.**

Altere um arquivo YAML → métricas Prometheus, regras de alerta, configuração do Alertmanager e dashboards Grafana atualizam automaticamente em até 60 segundos. Sem configuração manual. Sem drift.

---

## O Problema

A maioria das organizações monitora sua infraestrutura mas não *observa* de verdade. Times configuram dashboards manualmente, colocam thresholds hardcoded, copiam regras Prometheus entre serviços e mantêm configs do Alertmanager na mão. Quando um target de SLO muda, alguém precisa atualizar o dashboard, a regra de alerta, o roteamento e o runbook — separadamente, torcendo pra nada ser esquecido.

Isso piora com escala. 20 serviços significam 20 conjuntos de dashboards, alertas e thresholds pra manter manualmente. Drift de configuração se torna inevitável. Responsabilização se torna impossível.

**OaaC trata configuração de observabilidade da mesma forma que o Terraform trata infraestrutura: como código, versionado, validado e automatizado.**

---

## Como Funciona

```
catalogs/*.yml (YAML)
    │
    ├─→ slo_exporter.py ──→ Prometheus /metrics (port 9099)
    │   Expõe: targets de SLO, budgets de latência, metadados de serviço
    │
    ├─→ generate_alerts.py (gerador unificado)
    │   ├──→ alerts.yml ──→ Prometheus Rules
    │   │    Regras de alerta a partir de AlertPolicy + templates
    │   └──→ alertmanager.yml ──→ Configuração do Alertmanager
    │        Receivers, rotas, inhibit rules, mute intervals a partir de Team
    │
    └─→ Dashboards Grafana
        JOINs PromQL entre métricas reais e thresholds catalog-driven
```

**Hot-reload (60s):** slo_exporter lê YAMLs → atualiza métricas Prometheus → regenera alerts.yml + alertmanager.yml → dispara reload no Prometheus e no Alertmanager. Zero intervenção manual.

---

## Estrutura do Catálogo

Cinco tipos de recurso declarativos, inspirados no modelo de recursos do Kubernetes:

```
catalogs/
├── services/       → kind: Service      (identidade, ownership, referências)
├── slos/           → kind: SLO          (targets de disponibilidade, budgets de latência)
├── alerts/         → kind: AlertPolicy  (regras PromQL, thresholds, runbooks)
├── teams/          → kind: Team         (membros, canais, receivers, roteamento, inhibit rules)
└── dependencies/   → kind: Dependency   (componentes compartilhados, criticidade)
```

### Service — O Recurso Central

```yaml
apiVersion: golabs.local/v1
kind: Service
metadata:
  name: payments-api
  displayName: "Payments API"
spec:
  tipo: aplicacao
  criticidade: alta
  owner: sre                    # → teams/sre.yml
  slo_ref: payments-api-slo     # → slos/payments-api-slo.yml
  alert_ref: payments-api-alerts # → alerts/payments-api-alerts.yml
  dependency_refs:
    - traefik
    - postgres-cluster
  metricas:
    traefik_service: "payments@docker"
    job_name: "payments-api:8000"
```

### SLO — Contratos de Confiabilidade

```yaml
apiVersion: golabs.local/v1
kind: SLO
metadata:
  name: payments-api-slo
  service: payments-api
spec:
  disponibilidade:
    target: "99.9%"
    janela: "7d"
    error_budget_policy: alertar_time
  latencia:
    p99: "500ms"
    p50: "100ms"
```

Mude `target: "99.9%"` para `target: "99.5%"` → em até 60 segundos, os thresholds dos gauges no Grafana, painéis de error budget e cálculos de burn rate refletem o novo target. Sem editar dashboard.

### Team — Roteamento e Governança

O kind Team controla toda a configuração do Alertmanager. Receivers, regras de roteamento, inhibit rules e mute intervals são definidos aqui — o `alertmanager.yml` é gerado, nunca editado manualmente.

```yaml
apiVersion: golabs.local/v1
kind: Team
metadata:
  name: sre
  displayName: "Site Reliability Engineering"
spec:
  membros:
    - nome: Ighor Pitta
      papel: owner

  canais:
    critica: "teams #incidentes"
    alta: "teams #incidentes"
    media: "teams #alertas"
    baixa: email

  receivers:
    - name: teams_incidentes
      type: webhook
      # url: "https://hooks.example.com/incidentes"
    - name: teams_alertas
      type: webhook
      # url: "https://hooks.example.com/alertas"

  routing:
    defaults:
      resolve_timeout: 5m
      group_by: ['alertname', 'service', 'severity']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 1h
    overrides:
      critical:
        receiver_ref: teams_incidentes
        repeat_interval: 5m
      warning:
        receiver_ref: teams_incidentes
        repeat_interval: 1h
      info:
        receiver_ref: teams_alertas
        repeat_interval: 4h
        mute_time_intervals: ['fora-horario-comercial']

  inhibit_rules:
    - name: critical_suprime_warning
      source_severity: critical
      target_severity: warning
      equal: ['alertname', 'service']
    - name: host_down_suprime_containers
      source_alertname: HostDown
      target_category: containers
    - name: app_down_suprime_sintomas
      source_alertname_regex: ".*Down$"
      target_category_regex: "latencia|saturacao"

  mute_intervals:
    - name: fora-horario-comercial
      times: [{start: "19:00", end: "08:00"}]
    - name: fins-de-semana
      weekdays: ['saturday', 'sunday']
```

O gerador lê isso e produz um `alertmanager.yml` completo com receivers, roteamento por severidade, sub-rotas por serviço (baseado na criticidade do Service), inhibit rules e mute intervals. Adicionar um novo time ou mudar roteamento é uma edição de YAML, não uma reescrita de config do Alertmanager.

### AlertPolicy — Alertas a partir do Catálogo

AlertPolicies referenciam templates de alerta reutilizáveis ao invés de embutir PromQL bruto. O gerador resolve variáveis (`traefik_service`, targets do SLO) a partir dos kinds Service e SLO no momento da geração, produzindo regras Prometheus limpas sem valores hardcoded.

```yaml
apiVersion: golabs.local/v1
kind: AlertPolicy
metadata:
  name: payments-api-alerts
  service: payments-api
spec:
  alerts:
    - template: error_rate
      severidade: alta
      threshold: 0.01
      runbook: "https://wiki.internal/kb/payments-errors"
      acao_imediata: "Checar logs da Payments API e últimos deploys"

    - template: latency_p99
      severidade: alta
      # threshold omitido — puxado automaticamente do target do SLO
      runbook: "https://wiki.internal/kb/payments-latencia"
```

Templates ficam em `alerts/templates/` como padrões PromQL reutilizáveis com variáveis do catálogo:

```yaml
# alerts/templates/error_rate.yml
name: error_rate
categoria: disponibilidade
promql: >
  rate(traefik_service_requests_total{code=~"5..",service="{{ traefik_service }}"}[{{ janela }}])
  /
  rate(traefik_service_requests_total{service="{{ traefik_service }}"}[{{ janela }}])
defaults:
  janela: "5m"
  for: "5m"
  operador: ">"
```

O gerador lê o AlertPolicy, carrega o template referenciado, resolve `{{ traefik_service }}` a partir do kind Service e threshold a partir do kind SLO, e gera regras de alerta nativas do Prometheus. Grupos de alertas de infraestrutura existentes (host, container) são preservados.

---

## Decisões de Arquitetura

Escolhas de design documentadas como ADRs. Decisões principais:

| ADR | Decisão | Justificativa |
|-----|---------|---------------|
| ADR-01 | SLI individual por serviço, sem SLO global | SLO global ponderado por tráfego mascara falhas de serviços críticos |
| ADR-02 | SLI Composto = min(request-based, disponibilidade) | Captura tanto erros visíveis ao usuário quanto indisponibilidade total |
| ADR-03 | Latência como indicador visual, não parte do SLI | Evita confundir performance com confiabilidade |
| ADR-04 | Dual-source SLI — APM + Traefik | Traefik para request-based, APM para sinais de aplicação |
| ADR-05 | Geração de alertas baseada em templates | Elimina duplicação de PromQL, mantém alertas sincronizados com o catálogo |
| ADR-06 | Gerador unificado — Alertmanager a partir do catálogo | Roteamento de alerta é inseparável da definição do alerta; uma edição YAML, um reload |

---

## Abordagem de Dashboards

Dois dashboards complementares, ambos catalog-driven:

### Dashboard Executivo
Visão de tela única para liderança e on-call leads. Mostra o pior SLI entre todos os serviços, consumo de error budget ranqueado por severidade, latência P99 contra targets do catálogo. Tudo derivado de métricas do catálogo — sem thresholds hardcoded no JSON do Grafana.

### Dashboard Operacional (4 Níveis SRE)
Estruturado seguindo a metodologia Google SRE:

- **Nível 1 — SLA/SLO/SLI:** SLI por serviço com indicadores OK/BREACH catalog-driven, barras de error budget, burn rate
- **Nível 2 — Golden Signals:** Tráfego, percentis de latência, taxas de erro, saturação
- **Nível 3 — Confiabilidade:** MTBF/MTTR, burn rate 1h/6h, evolução do SLI ao longo do tempo
- **Nível 4 — Infraestrutura:** CPU, memória, GC, network I/O, file descriptors

Thresholds nos Níveis 1 e 3 são derivados dinamicamente do catálogo via JOINs PromQL entre métricas do Traefik e `service_slo_availability_target`.

---

## O Padrão de JOIN

O Traefik usa `service="app@docker"`. O catálogo usa `service="payments-api"`. A ponte é a label `traefik_service` exposta pelo `slo_exporter`:

```promql
# Renomeia traefik_service → service para casar com a label do Traefik
label_replace(
  1 - service_slo_availability_target,
  "service", "$1", "traefik_service", "(.+)"
)
```

Isso habilita JOINs PromQL entre métricas reais de requisições e targets definidos no catálogo, tornando dashboards totalmente catalog-driven sem nenhum valor hardcoded.

---

## Stack

100% open source. Zero licença proprietária.

| Componente | Papel |
|------------|-------|
| Prometheus | Armazenamento de métricas, engine de alertas |
| Grafana | Visualização, dashboards |
| Alertmanager | Roteamento e notificação de alertas (config gerada a partir do catálogo) |
| Traefik | Reverse proxy, fonte de métricas de requisições |
| Elasticsearch + APM | Observabilidade a nível de aplicação |
| slo_exporter.py | Ponte catálogo → métricas Prometheus + orquestrador de hot-reload |
| generate_alerts.py | Gerador unificado: catálogo → alerts.yml + alertmanager.yml |

---

## Validação

`validate-catalog-driven.sh` executa 6 verificações automatizadas:

1. SLO Exporter respondendo em `:9099/metrics`
2. Métricas do catálogo existem no Prometheus (`service_slo_availability_target`)
3. Grupo de alertas do catálogo (`catalog_alerts`) carregado no Prometheus
4. Grupos de alertas de infraestrutura preservados (não sobrescritos)
5. Dashboard referencia métricas do catálogo (`configFromData` para thresholds dinâmicos)
6. Buckets do histograma do Traefik incluem `le="0.5"` para precisão do P50

---

## Quick Start

```bash
# Subir a stack completa
docker compose --profile full up -d

# Verificar métricas do catálogo
curl "http://localhost:9090/api/v1/query?query=service_slo_availability_target"

# Editar um SLO e observar a propagação
vi slo-exporter/catalogs/slos/payments-api-slo.yml
# Mudar target: "99.9%" → "95%"
# Aguardar 60s → verificar no Grafana: cores dos gauges, error budget, burn rate atualizam
```

---

## Adicionando um Novo Serviço (< 5 minutos)

1. Criar `services/<nome>.yml` (kind: Service)
2. Criar `slos/<nome>-slo.yml` (kind: SLO)
3. Criar `alerts/<nome>-alerts.yml` (kind: AlertPolicy)
4. Referenciar time e dependências existentes, ou criar novos
5. O exporter detecta mudanças automaticamente em até 60 segundos

Sem editar Grafana. Sem alterar config do Prometheus. Sem alterar roteamento do Alertmanager. Sem escrever regras de alerta.

---

## Contexto e Inspiração

Este projeto se baseia nos mesmos princípios de ferramentas como [OpenSLO](https://openslo.com/), [Sloth](https://sloth.dev/) e as funcionalidades de observability-as-code do Grafana 12. O foco aqui é na **metodologia de adoção** — como levar observabilidade catalog-driven para organizações que ainda fazem monitoramento tradicional, onde o gap não é tooling, mas processo e estrutura.

Veja o [ROADMAP.md](ROADMAP.md) para o plano de evolução e descrição detalhada das fases.

---

## Autor

**Ighor Pitta** — SRE Engineer

- GitHub: [@ighorpitta](https://github.com/ighorpitta)
- LinkedIn: [ighorpitta](https://www.linkedin.com/in/ighorpitta)