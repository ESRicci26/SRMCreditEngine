# SRMCreditEngine

Plataforma de Cessao de Credito Multimoedas (SRM Credit Engine) - Desafio Técnico para vaga de
Engenheiro de Software Senior.

> Implementacao do desafio descrito em `README_case_dev_srm-V1.md` (documento original do
> processo seletivo, mantido na raiz para referencia).

> **V2 (atual):** banco de dados migrado de SQLite para **PostgreSQL**, com paginacao nativa
> via `LIMIT/OFFSET` habilitada pelo dialect oficial do Hibernate para PostgreSQL (na V1, o
> SQLite exigia um dialect customizado sem suporte a paginacao nativa, documentado na secao
> "Historico de Versoes" ao final deste README).

---

## 1. Stack Tecnologica

| Camada          | Tecnologia |
|-----------------|------------|
| Linguagem       | Java 11 |
| Framework       | Spring Boot 2.7.18 (Web, Data JPA, Validation, Actuator, AOP) |
| Build           | Maven |
| Frontend        | Thymeleaf + Bootstrap 5 (layout responsivo) + JS vanilla |
| Banco de Dados  | PostgreSQL 16 (via Docker Compose ou instancia gerenciada) |
| Persistencia    | Spring Data JPA (Hibernate) + dialect oficial `org.hibernate.dialect.PostgreSQL10Dialect` |
| Relatorios      | `NamedParameterJdbcTemplate` (SQL nativo, 2 camadas, sem passar pelo service) |
| Documentacao API| springdoc-openapi (Swagger UI) |
| Resiliencia     | Resilience4j (`@Retry` + `@CircuitBreaker`) |
| Observabilidade | Spring Actuator + Micrometer (Prometheus), logs estruturados com correlation id |
| Testes          | JUnit 5, Mockito, AssertJ |
| CI              | GitHub Actions (build + testes + checkstyle) |

### Por que essa stack?

- **Java 11 + Spring Boot**: solicitado no desafio; tipagem forte e o ecossistema Spring sao
  adequados para ambiente financeiro (validacao declarativa, transacoes gerenciadas, DI madura).
- **Spring Boot 2.7.x** (em vez de 3.x): ultima serie LTS-like do Spring Boot que ainda roda em
  Java 11 nativamente (Boot 3.x exige Java 17+), preservando o requisito de Java 11.
- **PostgreSQL (V2)**: banco relacional maduro, com suporte completo a `ALTER TABLE`/foreign keys
  (ao contrario do SQLite da V1) e dialect oficial mantido pelo proprio Hibernate
  (`org.hibernate.dialect.PostgreSQL10Dialect`), sem necessidade de nenhum dialect customizado.
  Esse dialect ja traduz paginacao (`Pageable`/`Specification`) em `LIMIT n OFFSET m` real no SQL
  gerado - paginacao nativa "de fabrica", sem o fallback em memoria que a V1 precisava usar com
  SQLite (ver secao "Historico de Versoes").
- **Thymeleaf**: server-side rendering, simples de manter, sem necessidade de build de SPA
  separado, adequado ao escopo e ao prazo do desafio; combinado com Bootstrap 5 para
  responsividade e JS vanilla (fetch) para a simulacao em tempo real, sem acoplar a um framework
  JS pesado.

---

## 2. Como Rodar o Projeto

### Pre-requisitos
- JDK 11+
- Maven 3.8+ (ou usar o wrapper, se disponivel: `./mvnw`)
- Docker + Docker Compose (para o PostgreSQL local) - ou uma instancia PostgreSQL 12+ propria

### Rodando localmente

```bash
# 1. Sobe o PostgreSQL local (porta 5432, banco/usuario/senha ja configurados)
docker compose up -d postgres

# 2. Compila e roda a aplicacao (conecta no Postgres via as variaveis de ambiente abaixo,
#    que ja tem os mesmos valores padrao usados no docker-compose.yml)
mvn clean install
mvn spring-boot:run
```

A aplicacao sobe em `http://localhost:8080`. As tabelas sao criadas automaticamente na primeira
execucao (`spring.jpa.hibernate.ddl-auto=update`) e os dados de referencia (moedas, tipos de
recebivel, taxas) sao populados automaticamente pelo `SemeadorDados`.

Variaveis de ambiente de conexao (com os valores padrao usados no `docker-compose.yml`):

| Variavel | Padrao |
|---|---|
| `DB_HOST` | `localhost` |
| `DB_PORT` | `5432` |
| `DB_NAME` | `srm_credit` |
| `DB_USER` | `srm_user` |
| `DB_PASSWORD` | `srm_password` |

### Rodando via Docker Compose (aplicacao + banco)

```bash
docker compose up -d --build
```

Sobe o PostgreSQL e a aplicacao juntos, ja conectados entre si (ver `docker-compose.yml`).


### URLs principais

| Recurso | URL |
|---|---|
| Dashboard | http://localhost:8080/ |
| Painel do Operador | http://localhost:8080/operador |
| Grid de Transacoes | http://localhost:8080/transacoes |
| Swagger UI | http://localhost:8080/swagger-ui.html |
| OpenAPI JSON | http://localhost:8080/v3/api-docs |
| Actuator Health | http://localhost:8080/actuator/health |
| Metricas Prometheus | http://localhost:8080/actuator/prometheus |

### Rodando os testes

```bash
mvn test
```

---

## 3. Arquitetura

### 3.1 Backend - 3 camadas (dominio) + 2 camadas (relatorios)

```
controller/          -> camada de aplicacao (REST + Web/Thymeleaf)
  web/                  -> controllers Thymeleaf (Painel do Operador, Grid)
service/             -> camada de negocio (regras, transacoes ACID, orquestracao)
repository/           -> camada de persistencia (Spring Data JPA)
  RepositorioRelatorioExtrato -> excecao deliberada: SQL nativo, chamado direto pelo
                              ControladorRelatorio (2 camadas, requisito 6 do desafio)
domain/
  entity/             -> agregados JPA
  enums/
  strategy/           -> Strategy Pattern do motor de precificacao
dto/                  -> contratos de entrada/saida da API
exception/            -> tratamento centralizado de excecoes
client/               -> integracao (mock) com provedor externo de cambio, resiliente
config/               -> OpenAPI, seed de dados, filtro de correlation id
```

### 3.2 Strategy Pattern (Motor de Precificacao)

```
EstrategiaPrecificacao (interface)
  └── EstrategiaPrecificacaoAbstrata (formula comum: VP = ValorFace / (1 + taxaBase + spread)^prazo)
        ├── EstrategiaPrecificacaoDuplicataMercantil   -> usa o spread do produto sem ajuste
        └── EstrategiaPrecificacaoChequePreDatado      -> soma +0.30 p.p. a.m. se prazo > 60 dias
FabricaEstrategiaPrecificacao  -> resolve a estrategia correta a partir do CodigoTipoRecebivel
```

Isso desacopla completamente a **regra de risco** (o que cada estrategia decide sobre o spread
efetivo) do **motor de calculo** (a formula de valor presente em si, escrita uma unica vez em
`EstrategiaPrecificacaoAbstrata`), permitindo adicionar novos produtos sem alterar codigo existente
(Open/Closed Principle).

Se a operacao for cross-currency (moeda do titulo diferente da moeda de liquidacao), a conversao
cambial e aplicada **apos** o calculo do valor presente, usando a taxa mais recente disponivel
(`ServicoMoeda.resolverTaxaConversao`), conforme especificado no desafio.

### 3.3 ACID e Concorrencia

- Toda escrita de negocio (`ServicoTransacao.criar`, `ServicoTransacao.liquidar`) ocorre dentro
  de uma unica fronteira `@Transactional`.
- A liquidacao (`settle`) usa **optimistic locking** via `@Version` na entidade `Transaction`: se
  duas requisicoes tentarem liquidar a mesma transacao concorrentemente, a segunda falha com
  `409 Conflict` (`ExcecaoLiquidacaoConcorrente` / `OptimisticLockingFailureException`) em vez de
  sobrescrever silenciosamente o estado.
- Nenhuma liquidacao fica "pela metade": o valor liquidado, a taxa aplicada e o status mudam juntos
  na mesma transacao de banco.

### 3.4 Resiliencia

`ClienteTaxaCambioExterna` simula uma integracao externa de cotacoes de cambio que falha ~30% das
vezes. As chamadas sao protegidas por `@Retry` (com backoff exponencial) e `@CircuitBreaker`
(Resilience4j), com fallback automatico para a ultima taxa de cambio persistida localmente,
garantindo que uma falha do provedor externo nao derrube a precificacao.

### 3.5 Observabilidade

- **Logs estruturados** com `correlationId` por requisicao (`FiltroCorrelacaoRequisicao` + MDC),
  facilitando rastreamento ponta a ponta de uma operacao nos logs.
- **Metricas**: Spring Actuator + Micrometer, com endpoint `/actuator/prometheus` pronto para
  scraping por um Prometheus/Grafana.
- **Health checks**: `/actuator/health` com detalhes (banco de dados, disco, etc).

### 3.6 Tratamento de Excecoes

`ManipuladorGlobalExcecoes` (`@RestControllerAdvice`) centraliza o tratamento de erros, garantindo que
nenhuma excecao interrompa o fluxo de forma abrupta ou vaze stack traces ao cliente:

| Excecao | HTTP Status |
|---|---|
| `ExcecaoRecursoNaoEncontrado` | 404 |
| `ExcecaoNegocio` | 422 |
| `ExcecaoLiquidacaoConcorrente` / `OptimisticLockingFailureException` | 409 |
| `MethodArgumentNotValidException` (Bean Validation) | 400 |
| `PropertyReferenceException` (ex: campo de `sort` invalido) | 400 |
| `MethodArgumentTypeMismatchException` (parametro fora do formato esperado) | 400 |
| `IllegalArgumentException` | 400 |
| Qualquer outra excecao | 500 (log completo, mensagem generica ao cliente) |

---

## 4. Modelagem de Dados

Ver `/docs/ER-diagram.md` (diagrama entidade-relacionamento em Mermaid) e `/docs/DDL.sql`
(script DDL documental do schema PostgreSQL). Ver tambem `/docs/C4-diagrams.md` para os diagramas de
Contexto e Container (Nivel 1 e 2).

Decisoes de modelagem relevantes:
- Todo valor monetario e taxa usa `DECIMAL` com escala fixa (nunca `FLOAT`/`DOUBLE`), evitando
  erro de arredondamento em calculos financeiros.
- `exchange_rate` e `base_rate` sao tabelas *append-only* (nunca ha update in-place de uma
  cotacao), preservando o historico de qual taxa estava vigente em cada momento - importante
  para auditoria.
- `transaction_ledger` grava tambem os insumos do calculo (taxa base, spread, taxa de cambio
  aplicados), nao so o resultado, permitindo reconstruir/auditar qualquer operacao passada mesmo
  que as taxas de referencia mudem depois.
- **V2:** as chaves estrangeiras (`REFERENCES`) agora sao constraints reais, enforced pelo
  PostgreSQL - na V1 (SQLite), o dialect customizado desabilitava `ALTER TABLE` para viabilizar
  `ddl-auto=update`, entao as referencias eram apenas documentais no DDL, sem enforcement do banco.

---

## 5. Git Workflow

- **Hooks**: como o projeto e 100% Java/Maven (sem Node.js no build), os hooks foram
  implementados como scripts shell simples em `.githooks/` (em vez de Husky), rodando testes e
  o linter Checkstyle antes de commit/push. Para ativar localmente:
  ```bash
  git config core.hooksPath .githooks
  ```
- **Commits**: historico organizado por etapa logica (dominio -> persistencia -> API -> frontend
  -> observabilidade/resiliencia -> docs), sem commits "wip"/"fix" soltos (squash aplicado via
  `rebase -i` antes do merge para a `main`).
- **Tag**: a entrega final e marcada com a tag `v1.0.0` (Semantic Versioning).

---

## 6. Endpoints Principais da API

| Metodo | Endpoint | Descricao |
|---|---|---|
| GET | `/api/currencies` | Lista moedas cadastradas |
| GET | `/api/currencies/rates` | Lista historico de taxas de cambio |
| POST | `/api/currencies/rates` | Registra/atualiza uma taxa de cambio manualmente |
| POST | `/api/pricing/simulate` | Simula o valor presente de um recebivel (sem persistir) |
| POST | `/api/transactions` | Precifica e persiste a aquisicao de um recebivel (PENDING) |
| GET | `/api/transactions` | Lista transacoes com filtros dinamicos + paginacao server-side |
| GET | `/api/transactions/{id}` | Consulta uma transacao |
| POST | `/api/transactions/{id}/settle?version=` | Liquida uma transacao (protegido por optimistic locking) |
| GET | `/api/reports/extrato?start=&end=&assignor=&currency=` | Extrato de liquidacao analitico |

Documentacao interativa completa (com schemas de request/response) disponivel em
`/swagger-ui.html` com a aplicacao rodando.

---

## 7. Criterios de Aceite (Requisito 5.2 do desafio)

- **Usabilidade**: painel do operador com feedback de calculo em tempo real (debounce de 350ms),
  grid com filtros e paginacao server-side, mensagens de erro claras e nao-tecnicas na UI.
- **Seguranca**: validacao de entrada via Bean Validation em todos os endpoints de escrita;
  nenhuma excecao interrompe o processo abruptamente; credenciais de banco via variaveis de
  ambiente (`DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`), nao hardcoded.
- **Desempenho**: relatorios analiticos usam SQL nativo (bypass do ORM) para consultas de grande
  volume; paginacao (`Pageable`/`Specification`) e server-side e, na V2, traduzida em `LIMIT/OFFSET`
  nativo no SQL gerado pelo dialect PostgreSQL - nao carrega grids inteiros em memoria em nenhuma
  das duas camadas.
- **Escalabilidade**: camadas desacopladas (Strategy Pattern para novos produtos sem alterar
  codigo existente; resiliencia a falhas de integracao externa via circuit breaker).

---

## 8. Estrutura de Pastas

```
SRMCreditEngine/
├── docs/                     # C4, ER, DDL
├── .github/workflows/ci.yml  # Pipeline de CI
├── .githooks/                # Git hooks (pre-commit, pre-push)
├── src/main/java/com/javaricci/
│   ├── client/                # Integracao externa resiliente (FX)
│   ├── config/                # OpenAPI, seed, correlation id
│   ├── controller/             # REST + web/ (Thymeleaf)
│   ├── domain/entity|enums|strategy/
│   ├── dto/request|response/
│   ├── exception/
│   ├── repository/
│   └── service/
├── src/main/resources/
│   ├── templates/            # Thymeleaf
│   ├── static/css|js/
│   └── application.yml
├── src/test/java/com/javaricci/
├── AI_USAGE.md
├── Dockerfile
└── pom.xml
```

---

## 9. Convencao de Nomenclatura em Portugues

Todo o codigo Java (classes, metodos, campos, variaveis locais, constantes e comentarios/Javadoc)
foi escrito/traduzido em portugues. Duas excecoes deliberadas e documentadas no proprio codigo:

- **Metodos de override de frameworks** (ex: `doFilterInternal` do `FiltroCorrelacaoRequisicao`,
  `run` do `SemeadorDados`) permanecem com o nome exigido pelo contrato de interface/classe do
  Spring - traduzi-los quebraria o override.
- **Nomes fisicos do banco de dados** (`@Column(name = ...)`, `@Table(name = ...)` e o SQL nativo
  em `RepositorioRelatorioExtrato`) permanecem no schema original em ingles (ja documentado em
  `docs/DDL.sql`), ja que sao strings de schema/dados, nao identificadores Java.

Como os campos das entidades e DTOs foram renomeados, os metodos de consulta derivada do Spring
Data (`findByCodigo`, `findFirstByMoedaOrderByVigenteEmDesc`, etc.) e os nomes das propriedades
usados em `Sort`/`Specification` (ex: `"criadoEm"`, `"moedaFace.codigo"`) foram atualizados em
conjunto para continuar batendo com os nomes de campo reais - uma inconsistencia aqui quebraria a
aplicacao em tempo de execucao (nao e so um erro de compilacao). O JSON trocado com o frontend
(`operator.js`/`transactions.js`) tambem foi atualizado para usar as mesmas chaves em portugues
dos DTOs (ex: `valorFace`, `dataVencimento`, `moedaLiquidacao`).

---

## 10. Historico de Versoes

### V2 (atual) - Migracao para PostgreSQL

- **Banco de dados:** SQLite -> PostgreSQL 16. Removida a dependencia `sqlite-jdbc` e a classe
  `DialetoSQLite` (dialect Hibernate customizado, necessario na V1 porque o Hibernate 5.6 nao tem
  suporte oficial a SQLite). Na V2, usa-se o dialect oficial e mantido pelo proprio Hibernate,
  `org.hibernate.dialect.PostgreSQL10Dialect` - nenhum codigo de dialect proprio e mais necessario.
- **Paginacao nativa via LIMIT/OFFSET:** o dialect SQLite customizado da V1 deixava
  `getLimitHandler()` no padrao (`NoopLimitHandler`), ou seja, o Hibernate paginava em memoria via
  pos-processamento do `ResultSet`. O `PostgreSQL10Dialect` ja usa nativamente
  `LimitOffsetLimitHandler`, entao toda paginacao (`Pageable` na API/grid de transacoes,
  `Specification` nos filtros dinamicos) agora gera `LIMIT n OFFSET m` real no SQL, sem
  necessidade de nenhuma configuracao ou codigo adicional - foi so consequencia de trocar o
  dialect.
- **Foreign keys reais:** as constraints `REFERENCES` no schema agora sao enforced pelo banco
  (o SQLite da V1 tinha `hasAlterTable()`/`dropConstraints()` desabilitados no dialect customizado
  para viabilizar `ddl-auto=update`, entao as FKs eram apenas documentais).
- **Infraestrutura local:** adicionado `docker-compose.yml` com um servico PostgreSQL (e,
  opcionalmente, a propria aplicacao) para desenvolvimento local, substituindo o arquivo
  `srm_credit.db` que ficava na raiz do projeto na V1.
- **Configuracao via variaveis de ambiente:** `SRM_DB_PATH` (caminho do arquivo SQLite) foi
  substituida por `DB_HOST`/`DB_PORT`/`DB_NAME`/`DB_USER`/`DB_PASSWORD`, seguindo o padrao usual
  de conexao a um banco relacional externo.

### V1 - Entrega inicial (SQLite)

Primeira entrega do desafio, com SQLite como banco de dados (arquivo na raiz do projeto, conforme
solicitado no enunciado original) e um dialect Hibernate customizado (`DialetoSQLite`) para suprir
a ausencia de suporte oficial a SQLite no Hibernate 5.6.x.

---

## 11. Autoria e Uso de IA

Consulte `AI_USAGE.md` para detalhes de prompts utilizados, correcoes aplicadas a sugestoes
incorretas da IA e a analise critica do processo, conforme exigido na Politica de Uso de IA do
desafio (secao 2 do `README_case_dev_srm-V1.md`).

