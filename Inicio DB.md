
# TalentCore — Documentação do Banco de Dados e Arquitetura (MVP)

> Projeto: **TalentCore** – Banco de Talentos corporativo
>
> Autor: Jonas Mexilem Rodrigues da Silva (ANALISTA SOLUCOES I)
>
> Data: 2026-02-25

---

## 📌 Sumário
- [Visão Geral](#-visão-geral)
- [Arquitetura Adotada (Hexagonal)](#-arquitetura-adotada-hexagonal)
  - [Princípios](#princípios)
  - [Pacotes e Organização](#pacotes-e-organização)
  - [Fluxo de Requisição (exemplo POST /candidates)](#fluxo-de-requisição-exemplo-post-candidates)
- [Banco de Dados Oracle](#-banco-de-dados-oracle)
  - [Modelo Lógico (MVP)](#modelo-lógico-mvp)
  - [Diagrama ER (ASCII)](#diagrama-er-ascii)
  - [DDL — Tabela, Sequence, Triggers e Índices](#ddl--tabela-sequence-triggers-e-índices)
  - [Boas Práticas e Decisões de Design](#boas-práticas-e-decisões-de-design)
  - [Consultas Úteis](#consultas-úteis)
  - [Seeds (dados de teste)](#seeds-dados-de-teste)
  - [Rollback (limpeza do ambiente de dev)](#rollback-limpeza-do-ambiente-de-dev)
  - [Migrações (versionamento de scripts)](#migrações-versionamento-de-scripts)
- [Backend (Java 17, sem frameworks)](#-backend-java-17-sem-frameworks)
  - [Ports](#ports)
  - [Use Case](#use-case)
  - [Adapters](#adapters)
  - [Configuração de Conexão Oracle](#configuração-de-conexão-oracle)
  - [Execução do Servidor HTTP Nativo](#execução-do-servidor-http-nativo)
  - [Considerações sobre JDBC no Oracle](#considerações-sobre-jdbc-no-oracle)
- [Evoluções Futuras](#-evoluções-futuras)
- [Checklist de Qualidade](#-checklist-de-qualidade)
- [Changelog](#-changelog)

---

## 🚀 Visão Geral
O **TalentCore** é um sistema de **banco de talentos corporativo**. O MVP implementa o cadastro de candidatos, com persistência em **Oracle** e um backend **Java 17** sem frameworks, seguindo **Arquitetura Hexagonal (Ports & Adapters)**. O front é **Angular 17 (standalone)**.

**Objetivo do MVP**: habilitar **CRUD de candidatos**, começando por **Create** e **Read** (por ID e lista paginada), com auditoria básica (`CREATED_AT`/`UPDATED_AT`).

---

## 🏛️ Arquitetura Adotada (Hexagonal)

### Princípios
- **Domínio independente**: regras de negócio sem dependência de infraestrutura.
- **Ports**: interfaces que descrevem capacidades do sistema (entrada e saída).
- **Adapters**: implementações concretas para tecnologias (HTTP, Oracle, etc.).
- **Teste** e **evolução** facilitados por baixo acoplamento.

### Pacotes e Organização
```
src/
 ├─ domain/
 │   └─ candidate/
 │       ├─ Candidate.java
 │       └─ CandidateValidator.java
 ├─ application/
 │   ├─ ports/
 │   │   ├─ in/
 │   │   │   └─ CreateCandidateUseCase.java
 │   │   └─ out/
 │   │       └─ CandidateRepositoryPort.java
 │   └─ usecases/
 │       └─ CreateCandidateService.java
 ├─ adapters/
 │   ├─ in/
 │   │   └─ http/
 │   │       └─ HttpServerApp.java
 │   └─ out/
 │       └─ oracle/
 │           ├─ OracleConnectionFactory.java
 │           └─ OracleCandidateRepository.java
 └─ config/
     └─ AppConfig.java
```

### Fluxo de Requisição (exemplo `POST /candidates`)
1. **Adapter In (HTTP)** recebe JSON → extrai `fullName`, `email`, `phone`, `skills`.
2. Chama **Port In** (`CreateCandidateUseCase`).
3. **Use Case** valida, normaliza e consulta **Port Out** (`CandidateRepositoryPort`).
4. **Adapter Out (Oracle)** executa SQL (JDBC) e retorna a entidade criada.
5. **Adapter In** devolve a resposta JSON ao cliente.

---

## 🗄️ Banco de Dados Oracle

### Modelo Lógico (MVP)
- **CANDIDATE**: entidade principal de candidato.
- **Gerador de IDs**: `SEQ_CANDIDATE_ID` + trigger `TRG_CANDIDATE_ID` (atribui ID no INSERT quando `NULL`).
- **Auditoria**: trigger `TRG_CANDIDATE_UPDATED_AT` (preenche `UPDATED_AT` em qualquer UPDATE). `CREATED_AT` com `DEFAULT SYSTIMESTAMP` e `NOT NULL`.
- **Índice função** em `LOWER(EMAIL)` para buscas **case-insensitive**.

### Diagrama ER (ASCII)
```
+-------------------------------+
|           CANDIDATE           |
+-------------------------------+
| ID            : NUMBER (PK)   |
| FULL_NAME     : VARCHAR2(150) |
| EMAIL         : VARCHAR2(150) |
| PHONE         : VARCHAR2(20)  |
| SKILLS        : CLOB          |
| CREATED_AT    : TIMESTAMP NN  |
| UPDATED_AT    : TIMESTAMP     |
+-------------------------------+
Indexes:
  - UQ_CANDIDATE_EMAIL (UNIQUE EMAIL)
  - IDX_CANDIDATE_EMAIL_LOWER (LOWER(EMAIL))
  - IDX_CANDIDATE_FULL_NAME (FULL_NAME) [opcional]
Triggers:
  - TRG_CANDIDATE_ID (before insert → ID = SEQ_CANDIDATE_ID.nextval)
  - TRG_CANDIDATE_UPDATED_AT (before update → UPDATED_AT = SYSTIMESTAMP)
```

### DDL — Tabela, Sequence, Triggers e Índices
> Execute esses scripts **no schema do seu usuário** (sem precisar criar novo usuário).

```sql
-- TABELA PRINCIPAL
CREATE TABLE CANDIDATE (
    ID           NUMBER        PRIMARY KEY,
    FULL_NAME    VARCHAR2(150) NOT NULL,
    EMAIL        VARCHAR2(150) NOT NULL UNIQUE,
    PHONE        VARCHAR2(20),
    SKILLS       CLOB,
    CREATED_AT   TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL,
    UPDATED_AT   TIMESTAMP
);

-- SEQUENCE PARA ID
CREATE SEQUENCE SEQ_CANDIDATE_ID START WITH 1 INCREMENT BY 1 NOCACHE;

-- TRIGGER DE ID AUTOMÁTICO
CREATE OR REPLACE TRIGGER TRG_CANDIDATE_ID
BEFORE INSERT ON CANDIDATE
FOR EACH ROW
BEGIN
    IF :NEW.ID IS NULL THEN
        :NEW.ID := SEQ_CANDIDATE_ID.NEXTVAL;
    END IF;
END;
/

-- TRIGGER DE AUDITORIA UPDATED_AT
CREATE OR REPLACE TRIGGER TRG_CANDIDATE_UPDATED_AT
BEFORE UPDATE ON CANDIDATE
FOR EACH ROW
BEGIN
    :NEW.UPDATED_AT := SYSTIMESTAMP;
END;
/

-- ÍNDICES
-- (1) índice para buscas case-insensitive por EMAIL
CREATE INDEX IDX_CANDIDATE_EMAIL_LOWER ON CANDIDATE (LOWER(EMAIL));

-- (2) índice para filtro por nome (prefixo)
CREATE INDEX IDX_CANDIDATE_FULL_NAME ON CANDIDATE (FULL_NAME);
```

### Boas Práticas e Decisões de Design
- **EMAIL em minúsculas**: normalizar no backend e usar `LOWER(EMAIL)` nas consultas.
- **UNIQUE(EMAIL)**: integridade lógica para evitar duplicidades.
- **DEFAULT + NOT NULL em CREATED_AT**: garante carimbo na criação mesmo sem o app preencher.
- **Triggers simples**: regras técnicas (ID e timestamps) ficam no DB — o domínio permanece limpo.
- **Clob para SKILLS (MVP)**: flexível para texto livre. Para filtros avançados, normalizar em tabelas `SKILL` e `CANDIDATE_SKILL` depois.
- **Paginação**: usar `OFFSET ... ROWS FETCH NEXT ... ROWS ONLY` (Oracle 12c+).

### Consultas Úteis
```sql
-- Buscar por ID
SELECT * FROM CANDIDATE WHERE ID = :id;

-- Buscar por EMAIL (case-insensitive)
SELECT * FROM CANDIDATE WHERE LOWER(EMAIL) = LOWER(:email);

-- Listagem paginada
SELECT ID, FULL_NAME, EMAIL, PHONE
FROM CANDIDATE
ORDER BY ID
OFFSET :offset ROWS FETCH NEXT :limit ROWS ONLY;

-- Conferir índices
SELECT INDEX_NAME, TABLE_NAME, UNIQUENESS FROM USER_INDEXES WHERE TABLE_NAME = 'CANDIDATE';
```

### Seeds (dados de teste)
```sql
INSERT INTO CANDIDATE (FULL_NAME, EMAIL, PHONE, SKILLS)
VALUES ('Teste Trigger', 'teste.trigger@example.com', '11970001122', 'Java, Angular, Oracle');

INSERT INTO CANDIDATE (FULL_NAME, EMAIL, PHONE, SKILLS)
VALUES ('Maria Dev', 'maria.dev@example.com', '11977776666', 'Java 17; Arquitetura Hexagonal; Oracle');

INSERT INTO CANDIDATE (FULL_NAME, EMAIL, PHONE, SKILLS)
VALUES ('Carlos Front', 'carlos.front@example.com', '21966665555', 'Angular 17; RxJS; Tailwind');

COMMIT;
```

### Rollback (limpeza do ambiente de dev)
```sql
BEGIN EXECUTE IMMEDIATE 'DROP INDEX IDX_CANDIDATE_FULL_NAME'; EXCEPTION WHEN OTHERS THEN NULL; END; /
BEGIN EXECUTE IMMEDIATE 'DROP INDEX IDX_CANDIDATE_EMAIL_LOWER'; EXCEPTION WHEN OTHERS THEN NULL; END; /
BEGIN EXECUTE IMMEDIATE 'DROP TRIGGER TRG_CANDIDATE_UPDATED_AT'; EXCEPTION WHEN OTHERS THEN NULL; END; /
BEGIN EXECUTE IMMEDIATE 'DROP TRIGGER TRG_CANDIDATE_ID'; EXCEPTION WHEN OTHERS THEN NULL; END; /
BEGIN EXECUTE IMMEDIATE 'DROP TABLE CANDIDATE CASCADE CONSTRAINTS'; EXCEPTION WHEN OTHERS THEN NULL; END; /
BEGIN EXECUTE IMMEDIATE 'DROP SEQUENCE SEQ_CANDIDATE_ID'; EXCEPTION WHEN OTHERS THEN NULL; END; /
```

### Migrações (versionamento de scripts)
Estruture no repositório uma pasta `db/migrations`:
```
db/
  migrations/
    V001__create_candidate.sql
    V002__seed_candidate.sql
    V010__add_indexes.sql
```

> Em produção, considere **Flyway** (CLI) para aplicar em ordem. No MVP, execução manual é suficiente.

---

## 💻 Backend (Java 17, sem frameworks)

### Ports
- **IN**: `CreateCandidateUseCase` (define a operação disponível ao mundo externo).
- **OUT**: `CandidateRepositoryPort` (contrato de persistência, implementado por Oracle JDBC).

### Use Case
- `CreateCandidateService`:
  - Valida `fullName`/`email`, normaliza e-mail para minúsculas.
  - Verifica duplicidade `existsByEmail`.
  - Chama `repository.create`.

### Adapters
- **IN/HTTP**: `HttpServerApp` com `com.sun.net.httpserver.HttpServer`.
- **OUT/Oracle**: `OracleCandidateRepository` com `PreparedStatement`, tratamento de `UNIQUE(EMAIL)` e mapeamento `TIMESTAMP → OffsetDateTime`.

### Configuração de Conexão Oracle
Variáveis de ambiente (ou `-D` system properties):
- `TALENTCORE_DB_URL` → `jdbc:oracle:thin:@//HOST:1521/SERVICE_NAME`
- `TALENTCORE_DB_USER` → *seu usuário/schema*
- `TALENTCORE_DB_PASS` → *sua senha*

### Execução do Servidor HTTP Nativo
```bash
# Compilar
javac -d out $(find src -name "*.java")

# Executar (defina as variáveis antes)
java -cp out br.com.talentcore.adapters.in.http.HttpServerApp
```
Teste rápido:
```bash
curl -X POST http://localhost:8080/candidates \
  -H "Content-Type: application/json" \
  -d '{"fullName":"Ana Teste","email":"ana.teste@example.com","phone":"11999998888","skills":"Java, Angular"}'
```

### Considerações sobre JDBC no Oracle
- `getGeneratedKeys()` pode não retornar a PK com alguns drivers Oracle; use **fallback** com `SELECT SEQ_CANDIDATE_ID.CURRVAL FROM DUAL` após o insert.
- Utilize `PreparedStatement` para evitar SQL Injection.
- Paginação: `OFFSET ? ROWS FETCH NEXT ? ROWS ONLY` (Oracle 12c+).

---

## 🔭 Evoluções Futuras
- **Normalização de SKILLS** (`SKILL`, `CANDIDATE_SKILL`) e **EXPERIENCE** (1:N) com FKs e cascata.
- **Busca por skills** (JOINs e índices em colunas de relacionamento).
- **Endpoints adicionais**: `GET /candidates/:id`, `GET /candidates?limit&offset`, `PUT`, `DELETE`.
- **Validador de e-mail** mais robusto e logs estruturados.
- **Camada de DTOs** e parser JSON dedicado.

---

## ✅ Checklist de Qualidade
- [x] Tabela `CANDIDATE` criada
- [x] `SEQ_CANDIDATE_ID` + `TRG_CANDIDATE_ID`
- [x] `TRG_CANDIDATE_UPDATED_AT`
- [x] `CREATED_AT` com default + NOT NULL (ajustado)
- [x] Índice `IDX_CANDIDATE_EMAIL_LOWER`
- [x] Dados de seed
- [x] Ports & Adapters definidos

---

## 📝 Changelog
- **V0.1.0 (2026-02-25)**: MVP do banco; documentação inicial; estrutura Hexagonal; JDBC Oracle; endpoints iniciais planejados.

