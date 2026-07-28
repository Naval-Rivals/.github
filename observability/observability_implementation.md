<div align="center">
  <img src="https://avatars.githubusercontent.com/u/296882315?s=400&u=c45dc755f6cdd049b8e57e3adf220b7e456944e2&v=4" width="120"/>
</div>

# Implementação de Observabilidade

**Métricas, Logs, Traces e Melhorias**

**Naval Rivals Multiplayer Online**

Autor: Caio de Souza<br>
Data: 27 de julho de 2026

---

## Sumário

1. [Introdução](#1-introdução)
2. [Ferramentas Utilizadas](#2-ferramentas-utilizadas)
3. [Implementação da Observabilidade](#3-implementação-da-observabilidade)
   - 3.1 [Métricas](#31-métricas)
   - 3.2 [Traces distribuídos](#32-traces-distribuídos)
   - 3.3 [Logs](#33-logs)
4. [Dashboards](#4-dashboards)
   - 4.1 [Visão Geral](#41-visão-geral)
   - 4.2 [HTTP](#42-http)
   - 4.3 [Banco](#43-banco)
   - 4.4 [Traces](#44-traces)
5. [Problemas Encontrados](#5-problemas-encontrados)
6. [Melhorias Implementadas](#6-melhorias-implementadas)

---

## 1. Introdução

O objetivo deste documento é apresentar a implementação de observabilidade na aplicação, 
permitindo monitorar métricas, traces e desempenhos em tempo de execução. 
Além disso, são apresentadas melhorias de resiliência e qualidade baseadas nas informações coletadas. 

## 2. Ferramentas Utilizadas

| Ferramenta | Finalidade |
|------------|------------|
| Spring Boot Actuator | Exposição das Métricas |
| Micrometer | Instrumentação |
| Prometheus | Coleta de Métricas |
| Grafana | Dashboards |
| OpenTelemetry | Traces distribuídos |
| Jaeger | Visualização de traces |
| Loki | Agregação e armazenamento de logs |
| Logback | Framework de logging (via SLF4J) |
| Docker Compose | Infraestrutura |

## 3. Implementação da Observabilidade

### 3.1 Métricas

As métricas foram implementadas utilizando o **Spring Boot Actuator** em conjunto com o **Micrometer**, permitindo a exposição de informações da aplicação em tempo de execução através do endpoint `/actuator/prometheus`.

O Prometheus foi configurado como responsável pela coleta periódica dessas métricas, realizando o scraping dos dados expostos pela aplicação. Posteriormente, o Grafana foi utilizado para criação de dashboards de visualização e análise dos indicadores coletados.

Foram monitoradas métricas relacionadas à aplicação, requisições HTTP, JVM e banco de dados, permitindo identificar comportamentos anormais, gargalos de desempenho e possíveis problemas de disponibilidade.

Seguindo a arquitetura:

```
Spring Boot Actuator
        ↓
     Micrometer
        ↓
 /actuator/prometheus
        ↓
   Prometheus
        ↓
    Grafana
```

### 3.2 Traces distribuídos

Os traces distribuídos foram implementados utilizando o OpenTelemetry, permitindo acompanhar o fluxo completo de execução de cada requisição realizada na API.

Os dados coletados pelo OpenTelemetry são enviados para o Jaeger, responsável pelo armazenamento e visualização dos traces. Através da interface do Jaeger é possível analisar o caminho percorrido por uma requisição, o tempo gasto em cada etapa e identificar pontos de lentidão ou falhas durante a execução.

A instrumentação automática fornecida pelo OpenTelemetry permitiu capturar informações das requisições HTTP e das operações JDBC sem a necessidade de instrumentação manual em cada método da aplicação.

A arquitetura utilizada para coleta dos traces é apresentada a seguir:

```
Cliente
    ↓
Spring Boot
    ↓
OpenTelemetry
    ↓
Jaeger
```

### 3.3 Logs

Os logs da aplicação foram centralizados utilizando o Logback para geração das mensagens e o Loki para armazenamento e indexação. A visualização e consulta dos logs foram realizadas através do Grafana, permitindo correlacionar eventos com métricas e traces da aplicação.

A arquitetura utilizada para coleta e visualização dos logs é apresentada a seguir:

```
Spring Boot
    ↓
Logback
    ↓
Loki
    ↓
Grafana
```

## 4. Dashboards

### 4.1 Visão Geral

### 4.2 HTTP

### 4.3 Banco

### 4.4 Traces

## 5. Problemas Encontrados

### 5.1 Latência elevada em endpoints autenticadas

**Causa raiz:** Latência de rede ao bando de dados remoto (Supabase) combinada com queries obrigatórias no filtro de segurança

Através dos traces coletados no Jaeger, identificou-se que cada query ao PostgreSQL hospedado no Supabase leva em
média ~155-163ms de latência de rede, independentemente da complexidade da consulta. O problema se agrava porque o
SecurityFilter executa 2 queries em toda request autenticada antes de chegar no controller:

1. SELECT ... FROM users WHERE email=? — para validar o token JWT (~161ms)
2. SELECT ... FROM stats WHERE user_id=? — carregada automaticamente pelo JPA devido ao relacionamento @OneToOne com
fetch EAGER na entidade User (~155ms)

Isso significa que toda request autenticada já consome ~316ms apenas na autenticação, antes de executar qualquer
lógica de negócio.

## 6. Melhorias Implementadas
