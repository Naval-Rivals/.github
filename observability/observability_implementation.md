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
   - 5.1 [Latência elevada em endpoints autenticadas](#51-latência-elevada-em-endpoints-autenticadas)
   - 5.2 [Conexões invalidadas no pool (HikariCP)](#52-conexões-invalidadas-no-pool-(hikaricp))
   - 5.3 [Timeout em conexões SSE do Lobby](#53-timeout-em-conexões-sse-do-lobby)
   - 5.4 [Serialização instável do PageImpl no endpoint de ranking](#54-seriallização-instável-do-pageimpl-no-endpoint-de-ranking)
   - 5.5 [Endpoints com latência elevada](#55-endpoints-com-latência-elevada)
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

### 5.2 Conexões invalidadas no pool (HikariCP)

**Causa raiz:** O banco de dados remoto encerra conexões ociosas antes do tempo configurado no pool de conexões da aplicação.

Nos logs da aplicação foram identificados warnings recorrentes do HikariCP:

```
HikariPool-1 - Failed to validate connection org.postgresql.jdbc.PgConnection@...
  (This connection has been closed.). Possibly consider using a shorter maxLifetime value.
```

O HikariCP utiliza valores padrão de maxLifetime (30 minutos) e não possui keepalive-time configurado. O Supabase, no
entanto, encerra conexões inativas em um intervalo menor. Isso faz com que o pool mantenha referências a conexões já fechadas
pelo servidor, resultando em falhas de validação e necessidade de criar novas conexões sob demanda — adicionando latência
extra às requisições.


### 5.3 Timeout em conexões SSE do Lobby

**Causa raiz:** O endpoint SSE (/lobby/events) não possuí configuração de timeout adequada, fazendo com que conexões ociosas sejam encerradas pelo Tomcat.

Nos logs foram identificados erros de `AsyncRequestTimeoutException` seguidos de `HttpMediaTypeNotAcceptableException`.
Isso ocorre porque, ao expirar a conexão SSE, o Spring tenta tratar a exceção pelo `GlobalExceptionHandler` que retorna
*application/json* — incompatível com o content-type *text/event-stream* da conexão original. O resultado é um ciclo de
exceções que gera logs de warn/error desnecessários.

### 5.4 Serialização instável do PageImpl no endpoint de ranking

**Causa raiz:** O endpoint de ranking retorna diretamente um PageImpl do Spring Data sem utilizar PagedModel, gerando umwarning sobre a instabilidade da estrutura JSON retornada entre versões do framework.

### 5.5 Endpoints com latência elevada

| Método | Endpoint | Tempo |
---------|----------|-------|
| POST | /rooms | 1s |
| POST | /rooms/join | 1s |
| GET | /games/{gameId}/result | 800ms |
| DELETE | /rooms/{roomId} | 900ms |
| PATCH | /users/me/nickname | 1s |
| GET | /users/me/matches | 1.5s |
| GET | /users/me | 999ms |

## 6. Melhorias Implementadas

### 6.1 Configuração do pool de conexões HikariCP

**Problema relacionado:** [5.2](#52-conexões-invalidadas-no-pool-(hikaricp))

O pool de conexões utilizava configurações padrão do HikariCP (maxLifetime de 30 min, sem keepalive). O Supabase
encerra conexões ociosas em intervalos menores, resultando em conexões inválidas no pool e warnings recorrentes nos
logs.

**Solução:** Configuração explícita dos parâmetros do HikariCP no `application.yaml`

```yml
spring:
    datasource:
      hikari:
        maximum-pool-size: 10
        minimum-idle: 2
        max-lifetime: 600000        # 10 minutos (menor que o timeout do Supabase)
        keepalive-time: 120000      # Keepalive a cada 2 minutos
        connection-timeout: 20000   # Timeout de 20s para obter conexão
        idle-timeout: 300000        # 5 minutos ocioso antes de fechar
        validation-timeout: 5000    # 5s para validar conexão
```

### 6.2 Tratamento de timeout em conexões SSE

**Problema relacionado:** [5.3](#53-timeout-em-conexões-sse-do-lobby)

Quando o timeout do SseEmitter expirava, o Spring lançava `AsyncRequestTimeoutException`. O `GlobalExceptionHandler`
capturava essa exceção pelo handler genérico Exception.class e tentava retornar uma resposta JSON — incompatível com o
content-type text/event-stream da conexão SSE, gerando `HttpMediaTypeNotAcceptableException` em cascata.

**Solução:**

- Adição de handler específico para AsyncRequestTimeoutException no GlobalExceptionHandler que retorna 204 No Content sem body (evitando conflito de media type)
- Configuração de spring.mvc.async.request-timeout no application.yaml alinhado com o timeout do SseEmitter (5 minutos)

```yml
# application.yaml
 spring:
    mvc:
      async:
        request-timeout: 310000  # 5min + 10s de margem
```

```java
// GlobalExceptionHandler.java
@ExceptionHandler(AsyncRequestTimeoutException.class)
public ResponseEntity<Void> handleAsyncTimeout(AsyncRequestTimeoutException e) {
    log.debug("[SSE] Conexão SSE expirou por timeout (comportamento esperado)");
    return ResponseEntity.noContent().build();
}
```

### 6.3 Estabilização da serialização do endpoint de ranking

**Problema relacionado:** [5.4](#54-seriallização-instável-do-pageimpl-no-endpoint-de-ranking)

O `RankingController` retornava diretamente um `Page<RankingResponse>` (instância de PageImpl), cuja estrutura JSON pode
mudar entre versões do Spring Data, gerando warnings de instabilidade.

**Solução:** Criação de um DTO wrapper (PageResponse<T>) com contrato explícito de serialização, desacoplando a resposta da API da implementação interna do Spring Data:

```java
 public record PageResponse<T>(
      List<T> content,
      int page,
      int size,
      long totalElements,
      int totalPages,
      boolean last
  ) {
      public PageResponse(Page<T> page) {
          this(
              page.getContent(),
              page.getNumber(),
              page.getSize(),
              page.getTotalElements(),
              page.getTotalPages(),
              page.isLast()
          );
      }
  }
```

### 6.4 Fetch LAZY no relacionamento User -> Stats

**Problema relacionado:** [5.1](#51-latência-elevada-em-endpoints-autenticadas)

O relacionamento `@OneToOne(mappedBy = "user")` na entidade `User` utiliza fetch EAGER por padrão no JPA. Isso força o
carregamento de Stats sempre que o User é carregado — inclusive no `SecurityFilter`, onde **Stats não é necessário**.

**Solução:** Adição de `fetch = FetchType.LAZY` no mapeamento:

```java
// User.java
@OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
private Stats stats;
```

### 6.5 Otimização da latência do endpoint GET /users/me/matches

**Problema relacionado:** [5.5](#55-endpoints-com-latência-elevada)

A análise dos traces no Jaeger identificou que o endpoint `GET /users/me/matches` realizava consultas adicionais para carregar as entidades `User` associadas aos relacionamentos `winner` e `loser` da entidade `GameResult`. Como esses relacionamentos estavam configurados com `FetchType.EAGER`, o Hibernate executava consultas extras para buscar os usuários referenciados, aumentando a latência da requisição e caracterizando um cenário de carregamento adicional de entidades que pode evoluir para um problema de N+1 quando há muitos adversários distintos.

Além disso, a consulta utilizada para buscar o histórico de partidas filtrava pelos campos `winner_id` e `loser_id`, que não possuíam índices específicos, tornando a busca menos eficiente conforme o volume de dados aumenta.

**Solução:** 
- Alteração dos relacionamentos `winner` e `loser` para `FetchType.LAZY`, evitando o carregamento automático dessas entidades em consultas que não necessitam dessas informações.
- Utilização de `JOIN FETCH` na consulta do repositório para carregar explicitamente os usuários apenas no endpoint que necessita desses dados, eliminando consultas adicionais durante a montagem da resposta.
- Criação de índices nas colunas `winner_id`, `loser_id` e `finished_at`, otimizando a busca e ordenação do histórico de partidas.

```java
// GameResult.java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "winner_id")
private User winner;

@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "loser_id")
private User loser;
```

```java
// GameResultRepository.java
@Query(
    value = """
        SELECT gr
        FROM GameResult gr
        JOIN FETCH gr.winner
        JOIN FETCH gr.loser
        WHERE gr.winner.id = :userId
           OR gr.loser.id = :userId
        ORDER BY gr.finishedAt DESC
        """,
    countQuery = """
        SELECT COUNT(gr)
        FROM GameResult gr
        WHERE gr.winner.id = :userId
           OR gr.loser.id = :userId
        """
)
Page<GameResult> findByUserId(UUID userId, Pageable pageable);
```

```sql
-- v7__add_index_from_game_result.sql
CREATE INDEX idx_game_results_winner_id
    ON game_results (winner_id);

CREATE INDEX idx_game_results_loser_id
    ON game_results (loser_id);

CREATE INDEX idx_game_results_finished_at
    ON game_results (finished_at DESC);
```

### 6.6 Otimização da latência do endpoint PATCH /users/me/nickname

FALTA IMPLEMENTAR

 Problema relacionado: 5.5 (#55-endpoints-com-latência-elevada)

  A análise do trace no Jaeger revelou que o endpoint PATCH /users/me/nickname consumia ~1.6s para uma operação trivial
  de atualização. O fluxo executava 4 queries sequenciais ao banco remoto (~200ms cada devido à latência de rede com o
  Supabase):

  1. SELECT ... FROM users WHERE email=? — SecurityFilter (autenticação JWT) — inevitável
  2. SELECT ... FROM users WHERE nickname=? LIMIT 1 — verificação de unicidade via existsByNickname
  3. SELECT ... FROM users WHERE id=? — reload do usuário via findById para obter entidade gerenciada
  4. SELECT ... FROM stats WHERE user_id=? — carregamento do Stats ao construir o UserResponse
  5. UPDATE users SET email=?, nickname=?, password=? WHERE id=? — dirty checking do Hibernate (full entity update)

  As queries #3 e #4 são desnecessárias: o findById pode ser substituído por getReferenceById (que retorna um proxy sem
  query), e o carregamento de Stats ocorre apenas porque o UserResponse inclui StatsResponse — informação irrelevante
  para a resposta de alteração de nickname.

  Solução:

  - Substituição de findById por getReferenceById para obter a entidade gerenciada sem executar SELECT adicional, já que
  apenas o nickname precisa ser atualizado e o ID é conhecido.
  - Utilização de @Modifying @Query com update direto via JPQL, eliminando o padrão read-modify-write do Hibernate e
  evitando o carregamento completo da entidade.
  - Criação de um DTO de resposta enxuto (NicknameResponse) que retorna apenas o nickname atualizado, sem necessidade de
  carregar Stats.

  // UserRepository.java — novo método com update direto
  @Modifying
  @Query("UPDATE User u SET u.nickname = :nickname WHERE u.id = :id")
  void updateNickname(@Param("id") UUID id, @Param("nickname") String nickname);

  // NicknameResponse.java
  public record NicknameResponse(
          String nickname
  ) {}

  // UserService.java — método otimizado
  @Transactional
  public NicknameResponse changeNickname(UpdateNicknameRequest data, User user) {

      if (userRepository.existsByNickname(data.nickname())) {
          log.warn("[USER] Alteração de nickname falhou — nickname já em uso: {}, userId={}", data.nickname(),
  user.getId());
          throw new UserAlreadyExistsException("Já existe um usuário com esse apelido");
      }

      userRepository.updateNickname(user.getId(), data.nickname());
      log.info("[USER] Nickname alterado — userId={}, de '{}' para '{}'", user.getId(), user.getNickname(),
  data.nickname());
      return new NicknameResponse(data.nickname());
  }

  // UserController.java — retorno com DTO enxuto
  @PatchMapping("/me/nickname")
  public ResponseEntity<NicknameResponse> changeNickname(
          @RequestBody @Valid UpdateNicknameRequest requestBody,
          @AuthenticationPrincipal User user
  ) {
      var response = userService.changeNickname(requestBody, user);
      return ResponseEntity.ok(response);
  }

  Com essa otimização, o endpoint elimina 2 queries (SELECT por ID + SELECT stats) e o full entity update, reduzindo de
  5 operações ao banco para 2 (verificação de unicidade + UPDATE direto). O tempo esperado do endpoint cai de ~1.6s para
  ~500ms (considerando a latência de rede com o Supabase: ~160ms por query no SecurityFilter + ~160ms existsByNickname +
  ~160ms UPDATE).
