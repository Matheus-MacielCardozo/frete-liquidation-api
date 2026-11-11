# Plataforma de Liquidação Contábil de Fretes de Carga
## Arquitetura e Especificação Técnica (Enterprise Edition - Spring Boot 4)

---

## 1. Visão Geral do Projeto e Arquitetura de Pastas

Esta aplicação foi desenhada seguindo os princípios de **Hexagonal Architecture (Ports and Adapters)** combinados com **Domain-Driven Design (DDD)**. O pacote `strategy` reside no **Domínio**, assegurando que as regras de precificação fiquem isoladas de frameworks e detalhes de infraestrutura, permitindo o uso seguro de `sealed interfaces` com `permits` no mesmo pacote.

```text
src/main/java/br/com/empresa/fretes
├── domain
│   ├── model            # Entidades ricas e Value Objects (Carga, CalculoFrete)
│   ├── enums            # Tipos de Carga (PADRAO, FRAGIL, REFRIGERADA, PERIGOSA)
│   ├── strategy         # Padrão Strategy para cálculo dinâmico de fretes (Regra de Negócio)
│   └── exceptions       # Exceções puras de domínio (ex: CargaPerigosaSemSeguroException)
├── application
│   ├── ports            # Interfaces (Contratos): in (UseCases) e out (Repositories)
│   └── usecases         # Orquestração do domínio (Implementação das portas 'in')
└── infrastructure
    ├── web              # Controllers REST, DTOs (Records), GlobalExceptionHandler
    ├── persistence      # Entidades JPA, Repositories Spring Data, Mappers
    └── config           # Configurações de Beans, OpenAPI/Swagger, Segurança
```

---

## 2. Stack Tecnológica (Enterprise LTS & Spring Boot 4)

* **Linguagem:** Java 21 LTS
* **Framework:** Spring Boot 4.1.2 (com Spring Framework 7+)
* **Documentação de API:** SpringDoc OpenAPI 3.1.1 (`springdoc-openapi-starter-webmvc-ui:3.1.1`)
* **Banco de Dados:** PostgreSQL 16+ (Hospedado no Supabase ou local via Docker)
* **Gerenciamento de Migrações:** Flyway (`spring-boot-starter-flyway`)
* **Build Tool:** Maven

---

## 3. Configuração do Sistema (`application.yml`)

Configuração otimizada para o ecossistema Spring Boot 4.1.2, unificando conexões seguras, validação de schema e observabilidade.

```yaml
spring:
  application:
    name: freight-accounting-platform

  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:freight_db}?sslmode=${DB_SSL_MODE:prefer}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
      connection-timeout: 30000

  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
    hibernate:
      ddl-auto: validate
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        highlight_sql: true
        use_sql_comments: true

  flyway:
    enabled: true
    baseline-on-migrate: true

server:
  port: 8080
  error:
    include-message: always
    include-binding-errors: always

springdoc:
  swagger-ui:
    path: /swagger-ui.html
  api-docs:
    path: /v3/api-docs

management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics
  endpoint:
    health:
      show-details: always
```

---

## 4. Contrato de API (REST)

### Endpoint: `POST /api/v1/fretes/calcular`

* **Cabeçalhos Obrigatórios (Headers):**
  * `Content-Type: application/json`
  * `X-Idempotency-Key: f47ac10b-58cc-4372-a567-0e02b2c3d479` *(Chave UUID única enviada exclusivamente no cabeçalho pelo cliente para prevenir duplicações de processamento/cobrança em retentativas de rede)*

**Request Payload (JSON):**
```json
{
  "origemCep": "63900-000",
  "destinoCep": "60000-000",
  "pesoKg": 125.50,
  "dimensoes": {
    "comprimentoCm": 120,
    "larguraCm": 80,
    "alturaCm": 60
  },
  "tipoCarga": "REFRIGERADA",
  "valorDeclaradoCents": 1500000
}
```
*(Nota: Valores monetários em centavos [Integer/Long] eliminam erros de arredondamento de ponto flutuante).*

**Response Payload (JSON) - `200 OK`:**
```json
{
  "liquidacaoId": "b82bb1c5-1234-4abc-99ef-0123456789ab",
  "idempotencyKey": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "custoBaseCents": 35000,
  "taxaAdicionalCents": 7500,
  "custoTotalCents": 42500,
  "distanciaEstimadaKm": 168.5,
  "prazoEstimadoDias": 2,
  "processadoEm": "2026-11-11T21:00:00Z"
}
```

---

## 5. Engenharia de Dados (Flyway Migration / DDL PostgreSQL)

Script oficial de migração (`src/main/resources/db/migration/V1__create_freight_tables.sql`):

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE freight_calculations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    idempotency_key UUID NOT NULL UNIQUE,
    origin_zip VARCHAR(9) NOT NULL,
    destination_zip VARCHAR(9) NOT NULL,
    weight_kg NUMERIC(10, 3) NOT NULL CHECK (weight_kg > 0),
    volume_m3 NUMERIC(10, 4) NOT NULL CHECK (volume_m3 > 0),
    cargo_type VARCHAR(20) NOT NULL,
    declared_value_cents BIGINT NOT NULL CHECK (declared_value_cents >= 0),
    base_cost_cents BIGINT NOT NULL,
    total_cost_cents BIGINT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT chk_cargo_type CHECK (cargo_type IN ('PADRAO', 'FRAGIL', 'REFRIGERADA', 'PERIGOSA'))
);

CREATE TABLE financial_audit_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    freight_id UUID NOT NULL,
    operation_type VARCHAR(50) NOT NULL,
    previous_value_cents BIGINT,
    new_value_cents BIGINT NOT NULL,
    triggered_by VARCHAR(100) DEFAULT 'SYSTEM',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT fk_audit_freight FOREIGN KEY (freight_id) 
        REFERENCES freight_calculations(id) ON DELETE RESTRICT
);

CREATE INDEX idx_freight_idempotency ON freight_calculations(idempotency_key);
CREATE INDEX idx_freight_cargo_type ON freight_calculations(cargo_type);
CREATE INDEX idx_audit_freight_id ON financial_audit_logs(freight_id);
```

---

## 6. Design Pattern: Strategy com Sealed Interfaces (Java 21 no Domínio)

*Localizado em `br.com.empresa.fretes.domain.strategy`.* Como as classes permitidas no `permits` precisam residir no mesmo pacote, todas ficam agrupadas no núcleo de domínio.

```java
package br.com.empresa.fretes.domain.strategy;

import br.com.empresa.fretes.domain.model.Carga;
import br.com.empresa.fretes.domain.model.ResultadoCalculo;

public sealed interface CalculoFreteStrategy 
    permits CalculoFretePadrao, CalculoFreteFragil, CalculoFreteRefrigerado, CalculoFretePerigoso {
    
    ResultadoCalculo calcular(Carga carga);
    
    default String getIdentificadorEstrategia() {
        return this.getClass().getSimpleName();
    }
}
```

---

## 7. Plano de Qualidade (QA) - Casos de Teste (JUnit 5 + Mockito)

1. **Caminho Feliz (Cálculo Completo de Carga Refrigerada):** Valida status HTTP 200, disparo da estratégia correta, persistência e auditoria financeira atômica.
2. **Idempotência de Requisição:** Envio duplicado com o mesmo header `X-Idempotency-Key` deve retornar o resultado cacheado/existente sem duplicar registros ou cobranças.
3. **Validação de Domínio (Bean Validation):** Payload com peso negativo ou CEP inválido interceptado antes do domínio, retornando HTTP 400 Bad Request.
4. **Cubagem vs. Peso Físico (Edge Case):** Verificação de preferência por peso cubado em cargas volumétricas leves.
5. **Transação Atômica e Rollback:** Simulação de falha no log de auditoria com Mockito para garantir o `@Transactional` e reverter a inserção principal.

---

## 8. DevOps, Conteinerização e CI/CD

### `Dockerfile` (Multi-stage Build com Maven 3.9.9 e Java 21)
```dockerfile
# Build Stage
FROM maven:3.9.9-eclipse-temurin-21-alpine AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Run Stage
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### `docker-compose.yml` (Ambiente Local Integrado)
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: freight_postgres
    environment:
      POSTGRES_DB: freight_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d freight_db"]
      interval: 5s
      timeout: 5s
      retries: 5

  app:
    build: .
    container_name: freight_api
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=freight_db
      - DB_USER=postgres
      - DB_PASSWORD=postgres
      - DB_SSL_MODE=disable
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  postgres_data:
```

### GitHub Actions Pipeline (`.github/workflows/ci.yml`)
```yaml
name: CI Pipeline - Freight Accounting Platform

on:
  push:
    branches: [ "main", "develop" ]
  pull_request:
    branches: [ "main", "develop" ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Java 21 LTS
        uses: actions/setup-java@v4
        with:
          distribution: 'temurin'
          java-version: '21'
          cache: 'maven'

      - name: Run Unit and Integration Tests
        run: mvn clean verify
```

---

## 9. Estrutura do `README.md` (Apresentação para Recrutadores)

```markdown
# 🚚 Freight Accounting Platform (API REST Corporativa)

[![Java 21 LTS](https://img.shields.io/badge/Java-21-orange.svg)](https://openjdk.org/projects/jdk/21/)
[![Spring Boot 4.1.2](https://img.shields.io/badge/Spring%20Boot-4.1.2-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-blue.svg)](https://www.postgresql.org/)
[![CI Status](https://github.com/SeuUsuario/frete-liquidation-api/actions/workflows/ci.yml/badge.svg)](https://github.com/SeuUsuario/frete-liquidation-api/actions/workflows/ci.yml)

Plataforma corporativa backend desenvolvida para liquidação contábil de fretes de carga, projetada com rigor de engenharia financeira, alta escalabilidade e padrões de arquitetura limpa.

## 🏛️ Destaques Arquiteturais & Engenharia
* **Arquitetura Hexagonal & DDD:** Domínio isolado de frameworks externos, garantindo testabilidade e manutenibilidade.
* **Design Pattern Strategy (Java 21 Sealed Interfaces no Domínio):** Regras de precificação polimórficas estritamente controladas em nível de compilação direto no núcleo de negócios (`br.com.empresa.fretes.domain.strategy`).
* **Idempotência Bancária:** Controle por cabeçalho (`X-Idempotency-Key`) para blindar o sistema contra cobranças duplicadas em falhas de rede.
* **Integridade Contábil:** Valores monetários manipulados em `Centavos (Long)` e migrações de banco controladas via **Flyway**.
* **Observabilidade & Documentação:** Endpoints de saúde integrados via **Spring Boot Actuator** e documentação interativa via **SpringDoc OpenAPI 3.1.1** (`/swagger-ui.html`).
```