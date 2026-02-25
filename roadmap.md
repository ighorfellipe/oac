# OaC Roadmap

Plano de evolução da observabilidade catalog-driven. Cada fase constrói sobre a anterior, avançando de protótipo funcional para metodologia production-grade.

---

## Estado Atual

O loop principal funciona end-to-end: catálogo YAML → métricas Prometheus → regras de alerta → config Alertmanager → dashboards Grafana, com hot-reload de 60 segundos. Dois serviços estão totalmente catalog-driven (dummy-app, podinfo) com dashboards executivo e operacional.

**O que está operacional:**

- 5 kinds no catálogo (Service, SLO, AlertPolicy, Team, Dependency)
- `slo_exporter.py` com hot-reload e resolução de referências entre todos os kinds
- `generate_alerts.py` unificado produzindo tanto `alerts.yml` (regras Prometheus) quanto `alertmanager.yml` (roteamento, receivers, inhibit rules, mute intervals)
- Alertmanager totalmente catalog-driven: receivers, roteamento por severidade, sub-rotas por serviço, inhibit rules, mute intervals — tudo a partir do kind Team
- Hot-reload dispara reload no Prometheus e no Alertmanager (POST `/-/reload` em ambos)
- Dashboard Executivo v11 (SLI individual, rankings, overlay de P99)
- Dashboard Operacional v8 (4 níveis SRE, Golden Signals, burn rate)
- SLI Composto: min(request-based, disponibilidade)
- Thresholds dinâmicos via JOIN PromQL (Traefik ↔ Catálogo)
- 6 verificações automatizadas de validação

**Limitação conhecida:** O gerador de alertas atualmente usa PromQL literal dos arquivos AlertPolicy sem resolver referências do catálogo. Nomes de serviço e thresholds estão hardcoded em cada AlertPolicy ao invés de serem derivados dos kinds Service e SLO. A Fase 1.5 resolve isso com um template engine.

---

## Fase 1 — Schema v2 e Thresholds de Infraestrutura

**Objetivo:** Eliminar todos os thresholds hardcoded dos dashboards Grafana.

Atualmente, serviços de infraestrutura (Grafana, Prometheus, Elasticsearch) usam thresholds fixos (0.97/0.999) no JSON do dashboard. Esta fase estende o catálogo para cobrir serviços de infra, para que todo threshold em todo dashboard venha de um arquivo YAML.

**Entregáveis:**
- Entradas no catálogo para serviços de infraestrutura (Grafana, Prometheus, Kibana, Elasticsearch, APM, Alertmanager)
- Painéis de dashboard para serviços de infra usando o mesmo padrão de JOIN dos serviços de aplicação
- Bump de versão do schema com compatibilidade retroativa

**Por que importa:** Enquanto qualquer threshold estiver hardcoded, a afirmação de "fonte única da verdade" tem um asterisco. Isso remove o asterisco.

---

## Fase 1.5 — Alert Template Engine

**Objetivo:** Eliminar PromQL hardcoded dos arquivos AlertPolicy. Alertas devem derivar labels de serviço e thresholds do catálogo, não duplicá-los.

**Problema sendo resolvido:** Hoje, cada AlertPolicy contém PromQL bruto com `service="app2@docker"` hardcoded e valores de threshold que duplicam o que já está nos kinds Service e SLO. Mudar o mapeamento Traefik de um serviço ou o target do SLO não propaga para seus alertas — exatamente o tipo de drift que o projeto existe para prevenir.

**Arquitetura:**

```
OaC/
├── catalog/
│   ├── loader.py              # Compartilhado: lê YAMLs, resolve referências
│   └── resolver.py            # Resolve variáveis de template a partir do catálogo
├── alerts/
│   ├── generator.py           # Orquestra: loader → templates → alerts.yml
│   └── templates/
│       ├── error_rate.yml     # Padrão PromQL reutilizável
│       ├── latency_p99.yml
│       ├── memory_high.yml
│       └── burn_rate_fast.yml # (Fase 5)
├── exporter/
│   └── slo_exporter.py        # Usa catalog.loader
└── catalogs/
```

**Entregáveis:**
- `catalog/loader.py` compartilhado, extraído da lógica de resolução de referências existente no slo_exporter
- Template engine que resolve `{{ traefik_service }}`, `{{ slo_target }}`, `{{ latency_p99 }}` a partir dos kinds do catálogo
- AlertPolicy simplificado para referências de template + overrides
- Output YAML limpo (sem newlines escapados ou strings multiline com aspas)
- Thresholds default por nível de criticidade (`defaults.yml`)

**AlertPolicy antes vs depois:**

```yaml
# ANTES — PromQL hardcoded, valores duplicados
alerts:
  - nome: error_rate_alta
    promql:
      expr: >
        rate(traefik_service_requests_total{code=~"5..",service="app2@docker"}[5m])
        / rate(traefik_service_requests_total{service="app2@docker"}[5m])
    threshold:
      operador: ">"
      valor: 0.01

# DEPOIS — referência a template, resolvido via catálogo
alerts:
  - template: error_rate
    severidade: alta
    threshold: 0.01
```

**Por que importa:** Esta é a diferença entre "YAML que gera alertas" e "governança de alertas catalog-driven". Sem isso, o pipeline de alertas está desconectado do catálogo — exatamente o problema que o OaC se propõe a resolver.

---

## Fase 2 — Validação e JSON Schema

**Objetivo:** Prevenir catálogos inválidos de serem aplicados.

**Entregáveis:**
- Definições JSON Schema para todos os 5 kinds
- `validate-catalog.py` que verifica conformidade com schema, integridade de referências (o `slo_ref` aponta para um SLO existente?) e presença de campos obrigatórios
- Pre-commit hook ou check de CI que bloqueia catálogos inválidos

**Por que importa:** Sem validação, um typo num arquivo YAML pode silenciosamente quebrar métricas ou alertas. Esta é a diferença entre um protótipo e um sistema confiável.

---

## Fase 3 — Pipeline CI/CD e GitOps

**Objetivo:** Workflow GitOps completo — Pull Request dispara validação, merge dispara deploy.

**Entregáveis:**
- Pipeline GitHub Actions: lint → validar schema → validar referências → dry-run generate → deploy
- Preview de PR: mostra quais métricas/alertas vão mudar antes do merge
- Rollback automático em caso de falha na validação
- Branch protection rules exigindo validação do catálogo

**Por que importa:** Aqui a metodologia se torna enforceável. Ninguém pode mudar um SLO sem um PR revisado. Toda mudança é auditável.

---

## Fase 4 — Dashboard Executivo Catalog-Driven ✅ CONCLUÍDA

**Objetivo:** Dashboard executivo onde todos os thresholds e indicadores vêm do catálogo.

**Entregue:**
- SLI individual por serviço com indicadores OK/BREACH catalog-driven
- Barras de error budget derivadas de `service_slo_availability_target`
- Latência P99 com overlay do target do SLO a partir do catálogo
- Rankings ordenados pelos piores performers
- Sem valores hardcoded no JSON do dashboard para serviços cobertos pelo catálogo

---

## Fase 5 — Alertas Multi-Window Multi-Burn Rate (MWMB)

**Objetivo:** Substituir alertas de threshold simples por alertas de burn rate no estilo Google SRE.

Atualmente, burn rate é exibido apenas no dashboard. Esta fase gera regras MWMB de alerta a partir do catálogo, seguindo o modelo descrito no Google SRE Workbook.

**Entregáveis:**
- Recording rules para razão de erro do SLI em múltiplas janelas (5m, 30m, 1h, 6h)
- Regras de alerta MWMB geradas a partir dos targets de SLO do catálogo
- Separação entre alertas de page (fast burn) e ticket (slow burn)
- Integração com roteamento do Alertmanager por severidade

**Por que importa:** Alertas de threshold simples ou disparam tarde demais (incidente perdido) ou com frequência demais (alert fatigue). MWMB é o padrão da indústria para alertas baseados em SLO e um forte sinal de maturidade SRE.

---

## Fase 6 — Onboarding Automatizado

**Objetivo:** Novo serviço totalmente observável em menos de 5 minutos.

**Entregáveis:**
- CLI ou gerador de template: `OaC new-service --name payments-api --tier 1 --team sre`
- Gera todos os arquivos do catálogo (Service, SLO, AlertPolicy) com defaults sensatos
- Opcional: geração automática de painel de dashboard a partir do catálogo
- Documentação: guia de onboarding para times com zero experiência em SRE

**Por que importa:** O teste definitivo da metodologia. Se adicionar um novo serviço à observabilidade completa leva 5 minutos e zero conhecimento de Grafana/Prometheus, a abstração funciona.

---

## Considerações Futuras

Estas não são fases comprometidas — são possibilidades que dependem de para onde o projeto evolui.

- **Governança SLO por tier:** Tier 0 (receita), Tier 1 (core), Tier 2 (suporte) com diferentes políticas de error budget por tier
- **Camada de compatibilidade OpenSLO:** Exportar catálogo para spec OpenSLO para interoperabilidade com Sloth, Nobl9 e outras ferramentas
- **Integração com Grafana 12 Git Sync:** Quando Git Sync amadurecer além de experimental, alinhar o workflow do catálogo com as funcionalidades nativas de as-code do Grafana
- **Atribuição de custos:** Mapear custos de observabilidade (storage, carga de queries) de volta para serviços e times do catálogo
- **Toggle enabled/disabled:** Capacidade de desabilitar o tracking de SLO de um serviço sem deletar a entrada no catálogo

---

## Registros de Decisão de Arquitetura (ADRs)

### ADR-01: SLI Individual por Serviço (Sem SLO Global)

**Status:** Aceito

**Contexto:** O design inicial considerou um SLO global ponderado por tráfego para o dashboard executivo.

**Decisão:** Cada serviço tem seu próprio SLI. A visão executiva mostra os piores performers via ranking, não agregação.

**Justificativa:** Um SLO global ponderado por tráfego mascara falhas em serviços críticos com baixo tráfego. Uma API de pagamento processando 100 req/s com SLI de 95% desaparece quando a média é feita com um site de docs fazendo 10.000 req/s a 99.99%. Rankings surfam problemas que a agregação esconde.

---

### ADR-02: SLI Composto = min(request-based, disponibilidade)

**Status:** Aceito

**Contexto:** SLI puramente request-based (razão de 5xx) não detecta indisponibilidade total — quando um container cai, não há requisições, então não há erros, e o SLI fica em 100%.

**Decisão:** SLI Composto é o mínimo entre SLI request-based e um sinal de disponibilidade baseado em tempo.

**Justificativa:** Isso captura tanto serviço degradado (alta taxa de erro sob carga) quanto falha completa (nenhuma requisição). A função `min()` garante que o SLI reflita o pior dos dois sinais.

---

### ADR-03: Latência como Indicador Visual, Não Parte do SLI

**Status:** Aceito

**Contexto:** Alguns frameworks de SLO incluem latência no cálculo do SLI composto.

**Decisão:** Latência (P99, P50) é exibida ao lado do SLI nos dashboards e tem targets definidos no catálogo, mas não afeta o número do SLI.

**Justificativa:** Misturar latência com SLI de disponibilidade confunde performance com confiabilidade. Um serviço pode estar lento mas disponível, ou rápido mas cheio de erros — essas situações exigem respostas diferentes. Mantê-los separados preserva a clareza do sinal. Targets de latência no catálogo servem como thresholds visuais (linhas de overlay nos gráficos) e podem disparar alertas separados.

---

### ADR-04: Dual-Source SLI — APM + Traefik

**Status:** Aceito

**Contexto:** A stack tem duas fontes de métricas de requisições: Traefik (reverse proxy) e Elastic APM (nível de aplicação).

**Decisão:** Usar ambos. Traefik para SLI request-based (ele vê todo o tráfego, incluindo requisições que nunca chegam à app). APM para diagnósticos e traces a nível de aplicação.

**Justificativa:** Traefik captura falhas que o APM perde (ex: app não respondendo, connection refused). APM fornece profundidade que o Traefik não consegue (traces de transação, queries de banco, detalhes de exceção). O catálogo referencia ambos via campos `traefik_service` e `job_name`.

---

### ADR-05: Geração de Alertas Baseada em Templates

**Status:** Aceito (evoluído do design inicial)

**Contexto:** O design inicial do AlertPolicy embutia expressões PromQL brutas com nomes de serviço hardcoded e thresholds. Isso criava drift — mudar o mapeamento `traefik_service` de um serviço ou o target do SLO não propagava para seus alertas. Também significava duplicar o mesmo padrão PromQL em cada serviço.

**Decisão:** AlertPolicies referenciam templates reutilizáveis. O gerador de alertas resolve variáveis do catálogo (labels de serviço, targets de SLO) no momento da geração, produzindo regras Prometheus completas.

**Justificativa:** Templates eliminam duplicação de PromQL entre serviços, garantem que alertas fiquem sincronizados com mudanças no catálogo, e mantêm arquivos AlertPolicy legíveis para revisores não-SRE. O trade-off é uma camada extra de indireção, mas o gerador resolve tudo no build time — Prometheus vê regras puras, sem abstração.

---

### ADR-06: Gerador Unificado — Config do Alertmanager a partir do Catálogo

**Status:** Aceito

**Contexto:** O `alertmanager.yml` era um arquivo estático de 23 linhas, mantido manualmente e desconectado do catálogo. Receivers, roteamento e inhibit rules não tinham relação com os kinds Team, Service ou AlertPolicy. Adicionar um novo serviço ou mudar roteamento por severidade exigia editar dois arquivos sem relação.

**Decisão:** Um único `generate_alerts.py` lê o catálogo uma vez e produz tanto `alerts.yml` (regras Prometheus) quanto `alertmanager.yml` (config completa do Alertmanager). O kind Team é a fonte primária para roteamento, receivers, inhibit rules e mute intervals. A `criticidade` do kind Service determina overrides de roteamento por serviço.

**Justificativa:** Roteamento de alertas é inseparável da definição de alertas — quem recebe o page é tão importante quanto o que dispara o page. Mantê-los em configs separadas e desconectadas garante drift. O gerador unificado garante que mudar os canais de um time, a criticidade de um serviço, ou uma inhibit rule seja uma única edição YAML com um único caminho de reload. O Alertmanager nunca precisa ser configurado diretamente.

---

## Glossário

| Termo | Definição |
|-------|-----------|
| **SLI** | Service Level Indicator — a confiabilidade medida de um serviço (ex: 99.7% das requisições bem-sucedidas) |
| **SLO** | Service Level Objective — o target de confiabilidade (ex: 99.9% de disponibilidade em 7 dias) |
| **SLA** | Service Level Agreement — compromisso contratual, geralmente com penalidades financeiras |
| **Error Budget** | A indisponibilidade permitida: `1 - target do SLO`. Com SLO de 99.9%, o budget é 0.1% |
| **Burn Rate** | Velocidade de consumo do error budget. Burn rate 1.0 = no ritmo. Acima de 1.0 = consumindo mais rápido que o permitido |
| **MWMB** | Multi-Window Multi-Burn — estratégia de alerta que usa múltiplas janelas de tempo e thresholds de burn rate para balancear velocidade de alerta com redução de ruído |
| **Golden Signals** | As quatro métricas-chave do Google SRE: latência, tráfego, erros, saturação |
| **Catalog-driven** | Configuração derivada de catálogos YAML em runtime, não hardcoded em dashboards ou arquivos de alerta |
| **Hot-reload** | Detecção e aplicação automática de mudanças no catálogo sem restart de serviço |
| **Kind** | Um tipo de recurso do catálogo (Service, SLO, AlertPolicy, Team, Dependency), inspirado no modelo de recursos do Kubernetes |
| **Alert Template** | Padrão PromQL reutilizável com variáveis do catálogo (ex: `{{ traefik_service }}`), resolvido no momento da geração em regras Prometheus nativas |
| **Gerador Unificado** | Script único que lê o catálogo e produz tanto regras de alerta do Prometheus quanto configuração do Alertmanager |
| **Inhibit Rule** | Regra do Alertmanager que suprime alertas de menor severidade quando um alerta de maior severidade já está disparando para o mesmo escopo |
| **Mute Interval** | Janela de tempo durante a qual certos alertas são silenciados (ex: fora do horário comercial, fins de semana) |