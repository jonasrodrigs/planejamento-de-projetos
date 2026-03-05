# TalentCore – Backend (Java 17 + Spring Boot 3)

**Arquitetura Hexagonal (Ports & Adapters) • DTOs limpos • Validação • Swagger • Perfis (MVP / Oracle)**

---

## Sumário
1. [Visão Geral](#visão-geral)
2. [Arquitetura & Pastas](#arquitetura--pastas)
3. [Fluxo de Requisições](#fluxo-de-requisições)
4. [Build, Execução & Perfis](#build-execução--perfis)
5. [Configurações & Ambiente](#configurações--ambiente)
6. [Modelo de Domínio](#modelo-de-domínio)
7. [Camada de Aplicação (Use Cases)](#camada-de-aplicação-use-cases)
8. [Adapters (HTTP & Persistence)](#adapters-http--persistence)
9. [DTOs, Mappers & Validação](#dtos-mappers--validação)
10. [Endpoints da API](#endpoints-da-api)
11. [Erros & Contratos de Resposta](#erros--contratos-de-resposta)
12. [Documentação com Swagger/OpenAPI](#documentação-com-swaggeropenapi)
13. [Testes](#testes)
14. [Observabilidade & Logs](#observabilidade--logs)
15. [Segurança (JWT – opcional)](#segurança-jwt--opcional)
16. [Banco de Dados Oracle (opcional agora)](#banco-de-dados-oracle-opcional-agora)
17. [Boas Práticas & Convenções](#boas-práticas--convenções)
18. [Troubleshooting (erros comuns)](#troubleshooting-erros-comuns)
19. [Roadmap & Próximos Passos](#roadmap--próximos-passos)
20. [Apêndice – Exemplos práticos](#apêndice--exemplos-práticos)

---

## Visão Geral
O **TalentCore** é um backend REST para gerenciamento de **candidatos** (perfis técnicos, experiências, projetos, idiomas e habilidades), implementado em **Java 17 + Spring Boot 3**, seguindo **Arquitetura Hexagonal** (Ports & Adapters).

O projeto foi pensado para:
- **MVP rápido** com repositório **InMemory** (perfil `mvp`);
- **Integração corporativa** com **Oracle** via JDBC puro (perfil `oracle`);
- **Domínio limpo** (POJOs sem dependência de framework);
- **Validações** no domínio e na borda (Bean Validation);
- **Documentação** com Swagger/OpenAPI.

---

## Arquitetura & Pastas
Estrutura por camadas (Hexagonal):

```
src/main/java/br/com/talentcore/talentos
├── TalentCoreApplication.java
├── api/                   # Adaptador de entrada (DTO + Mapper)
│   ├── dto/
│   │   ├── CandidatoRequest.java
│   │   └── CandidatoResponse.java
│   └── mapper/
│       └── CandidatoMapper.java
├── application/           # Casos de uso (Ports In/Out + Services)
│   ├── port/
│   │   ├── in/
│   │   │   ├── CadastrarCandidatoUseCase.java
│   │   │   ├── BuscarCandidatoUseCase.java
│   │   │   └── AtualizarCandidatoUseCase.java
│   │   └── out/
│   │       └── CandidatoRepository.java
│   ├── CadastrarCandidatoService.java
│   ├── BuscarCandidatoService.java
│   ├── AtualizarCandidatoService.java
│   └── CandidateBootstrapTest.java   # bootstrap manual (dev)
├── config/
│   ├── BeanConfig.java               # escolhe o adapter por profile
│   ├── DatabaseConfig.java           # JDBC manual (Oracle)
│   └── OpenApiConfig.java            # Swagger
├── domain/                 # Domínio puro (entidades, enums, serviços)
│   ├── service/
│   │   └── CandidatoService.java     # validação de agregados
│   ├── (entidades e enums ...)
│   └── ...
└── infrastructure/
    ├── http/              # Controller + Exception Handler
    │   ├── CandidatoController.java
    │   └── ApiExceptionHandler.java
    └── persistence/       # Adapters de persistência
        ├── CandidatoRepositoryOracle.java
        └── mvp/
            └── CandidatoRepositoryInMemory.java
```

### Diagrama (Mermaid)
```mermaid
flowchart LR
  A[HTTP Client (Swagger, Postman, Angular)] --> B[Controller / API DTO]
  B --> C[Use Cases (Application)]
  C --> D[Domain (Entities + Service)]
  C -->|Port Out| E{{CandidatoRepository}}
  E --> F[Adapter InMemory]
  E --> G[Adapter Oracle JDBC]
  G --> H[(Oracle DB)]
```

---

## Fluxo de Requisições
1. **Cliente** envia request → **Controller** recebe `DTO`.
2. **Mapper** converte `DTO → Domínio`.
3. **UseCase** orquestra: valida com `CandidatoService`, chama `Repository`.
4. **Repository** persiste/busca (InMemory/Oracle).
5. **Mapper** converte `Domínio → Response DTO`.
6. **Controller** retorna `ResponseEntity`.

---

## Build, Execução & Perfis

### Requisitos
- Java 17
- Maven 3.9+
- (Opcional) Oracle JDBC, se usar o perfil `oracle`

### Build
```bash
mvn clean package
```

### Rodar – **MVP (InMemory)**
`application.yml` já define `spring.profiles.active: mvp`.
```bash
mvn spring-boot:run
# ou
java -jar target/talentcore-1.0.0.jar
```

### Rodar – **Oracle**
Configure as variáveis e ative o profile:

**Windows PowerShell**
```powershell
$env:TC_DB_URL="jdbc:oracle:thin:@//HOST:1521/SERVICE"
$env:TC_DB_USER="TALENTCORE"
$env:TC_DB_PASSWORD="SENHA"
mvn spring-boot:run -Dspring-boot.run.profiles=oracle
```

**Linux/Mac**
```bash
export TC_DB_URL="jdbc:oracle:thin:@//HOST:1521/SERVICE"
export TC_DB_USER="TALENTCORE"
export TC_DB_PASSWORD="SENHA"
mvn spring-boot:run -Dspring-boot.run.profiles=oracle
```

Porta padrão: **8080**

Healthcheck:
```
GET http://localhost:8080/actuator/health
```

---

## Configurações & Ambiente

`src/main/resources/application.yml` possui dois perfis:

- **mvp (padrão)**: usa `CandidatoRepositoryInMemory`, desativa autoconfig do DataSource.
- **oracle**: usa `CandidatoRepositoryOracle` e lê `TC_DB_URL/USER/PASSWORD` via `DatabaseConfig`.

Actuator exposto: `/actuator/health`, `/actuator/info`.

---

## Modelo de Domínio
Principais entidades (POJOs puros):
- `Candidato` (agregado raiz)
- `Contato`, `Endereco` (Value Objects)
- `Experiencia`, `Projeto`, `Habilidade`, `Idioma` (filhos do agregado)
- Enums: `NivelConhecimento`, `NivelIdioma`, `TipoContratacao`, etc.

**Regras no `CandidatoService`:**
- `nomeCompleto` obrigatório.
- Garante **listas não nulas**.
- Coerência de **datas** (fim ≥ início).
- Normalização de **listas de strings** (trim, remove vazios).

---

## Camada de Aplicação (Use Cases)
- **CadastrarCandidatoUseCase** (`CadastrarCandidatoService`)
  - Define `id` (se vazio), valida agregado, **`existsByEmail`** e `salvar`.
- **BuscarCandidatoUseCase** (`BuscarCandidatoService`)
  - Encaminha filtros para `buscarPorFiltros`.
- **AtualizarCandidatoUseCase** (`AtualizarCandidatoService`)
  - Valida id, valida agregado, (opcional) regra de e-mail único, `atualizar(agregado)`.

**Port Out:** `CandidatoRepository`
- `existsByEmail(email)`
- `salvar(candidato)`
- `buscarPorId(id)`
- `buscarPorFiltros(...)`
- `atualizar(candidato)`

---

## Adapters (HTTP & Persistence)

### HTTP (infrastructure/http)
- `CandidatoController` com endpoints **POST, GET{id}, GET(filtros), PUT{id}**.
- `ApiExceptionHandler` padroniza erros: **400** (validação/argumento), **500** (genérico).
- Bean Validation ativada com `@Valid` nos métodos **POST/PUT**.

### Persistence
- **InMemory** (`mvp`): implementação funcional para desenvolvimento rápido.
- **Oracle** (`oracle`): **a implementar** com JDBC (DDL sugerido nesta doc). O `DatabaseConfig` abre `Connection` via variáveis de ambiente.

---

## DTOs, Mappers & Validação

### DTOs
- `CandidatoRequest`: entrada via POST/PUT (com Bean Validation).
- `CandidatoResponse`: saída “resumo” (id, nome, e-mail, cidade/estado/pais, habilidades, tecnologias, idiomas).

### Mapper
- `CandidatoMapper`
  - `fromRequest(req)` → domínio
  - `toResponse(domínio)` → resposta resumida
  - `merge(atual, req)` → **atualização parcial** (só substitui o que vier no payload)

### Bean Validation
- `@NotBlank`, `@NotNull`, `@Email`, `@Size`, `@Pattern` em `CandidatoRequest`.
- `ApiExceptionHandler` lida com `MethodArgumentNotValidException` e retorna `fields` com mensagens.

---

## Endpoints da API

Base URL: `http://localhost:8080`

### `POST /api/candidatos`
Cria um candidato.

**Body (exemplo):**
```json
{
  "nomeCompleto": "Jon Doe",
  "contato": { "telefone": "81-98888-0000", "email": "jon.doe@talentcore.dev" },
  "endereco": { "cidade": "RECIFE", "estado": "PE", "pais": "BRASIL" },
  "idiomas": [
    { "idioma": "PT-BR", "nivel": "NATIVO" },
    { "idioma": "EN-US", "nivel": "AVANCADO" }
  ],
  "habilidadesTecnicas": [
    { "nome": "Angular", "categoria": "Frontend", "nivel": "AVANCADO" },
    { "nome": "Java", "categoria": "Backend", "nivel": "INTERMEDIARIO" }
  ],
  "experiencias": [
    {
      "empresa": "Tech Co",
      "cargo": "Fullstack Dev",
      "tipo": "CLT",
      "dataInicio": "2024-01-01",
      "dataFim": "2025-01-01",
      "tecnologias": ["ANGULAR", "JAVA 17"]
    }
  ],
  "projetos": [
    {
      "nome": "Portal Vendas",
      "dataInicio": "2024-05-01",
      "dataFim": "2024-12-01",
      "tecnologias": ["ANGULAR", "RXJS"]
    }
  ]
}
```

**Resposta 200 (exemplo):**
```json
{
  "id": "b9f6...-...",
  "nomeCompleto": "Jon Doe",
  "email": "jon.doe@talentcore.dev",
  "cidade": "RECIFE",
  "estado": "PE",
  "pais": "BRASIL",
  "habilidades": ["Angular","Java"],
  "tecnologias": ["ANGULAR","JAVA 17","ANGULAR","RXJS"],
  "idiomas": ["PT-BR","EN-US"]
}
```

### `GET /api/candidatos/{id}`
Busca um candidato por id.

### `GET /api/candidatos`
Lista/filtra candidatos.

**Query params (todos opcionais):**
- `tecnologia` (ex.: `ANGULAR`)
- `nivel` (ex.: `AVANCADO`)
- `cidade` (ex.: `BARUERI`)
- `estado` (ex.: `SP`)
- `idioma` (ex.: `EN-US`)
- `nivelIdioma` (ex.: `FLUENTE`)
- `page` (padrão `0`), `size` (padrão `50`) — paginação simples em memória (MVP)

**Exemplos:**
```
GET /api/candidatos?cidade=BARUERI&estado=SP
GET /api/candidatos?tecnologia=ANGULAR
GET /api/candidatos?idioma=EN-US&nivelIdioma=FLUENTE
```

### `PUT /api/candidatos/{id}`
Atualiza **parcialmente** um candidato (merge).

**Body (exemplo):**
```json
{
  "contato": { "telefone": "81-97777-1234" },
  "endereco": { "cidade": "OLINDA" }
}
```

---

## Erros & Contratos de Resposta

### 400 – Validação
```json
{
  "timestamp": "2026-03-05T10:20:40-03:00",
  "status": 400,
  "error": "Bad Request",
  "message": "Validação falhou",
  "fields": {
    "nomeCompleto": "nomeCompleto é obrigatório",
    "contato.email": "must be a well-formed email address"
  }
}
```

### 404 – Não encontrado
Sem body, apenas status.

### 500 – Erro interno
```json
{
  "timestamp": "2026-03-05T10:20:40-03:00",
  "status": 500,
  "error": "Internal Server Error",
  "message": "Erro interno. Consulte os logs."
}
```

---

## Documentação com Swagger/OpenAPI
- **UI**: `http://localhost:8080/swagger-ui.html`  
  (ou `http://localhost:8080/swagger-ui/index.html`)
- **JSON**: `http://localhost:8080/v3/api-docs`

Dependência:
```xml
<dependency>
  <groupId>org.springdoc</groupId>
  <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
  <version>2.6.0</version>
</dependency>
```

---

## Testes
**Recomendado:**
- **Unitários (domínio):** `CandidatoServiceTest`
  - Datas (fim ≥ início), listas não nulas, normalização.
- **Unitários (mapper):** `CandidatoMapperTest`
  - fromRequest / toResponse / merge.
- **Integração (web):** `@WebMvcTest(CandidatoController.class)`
  - POST/GET/PUT com validação e exception handler.
- **Smoke** (profile `mvp`): subir app e chamar `/actuator/health`.

---

## Observabilidade & Logs
- **Actuator** habilitado: `/actuator/health`, `/actuator/info`.
- Configurar `logback-spring.xml` (opcional) com padrão JSON/MDC (`requestId`).
- Em produção, capturar logs com stack (ELK/EFK).

---

## Segurança (JWT – opcional)
Futuro próximo (quando o front Angular estiver pronto):
- **Interceptador JWT** no back (`Bearer <token>`).
- `SecurityFilterChain` permitindo `/api/candidatos/**` autenticado e `/actuator/health` público.
- Perfis de acesso: `ADMIN`, `RECRUTADOR`.

---

## Banco de Dados Oracle (opcional agora)
- **DDL** proposto nesta arquitetura (tabelas: `CANDIDATO`, `IDIOMA`, `HABILIDADE`, `EXPERIENCIA`, `EXP_TECNOLOGIA`, `PROJETO`, `PROJ_TECNOLOGIA`), com FK `ON DELETE CASCADE`.
- `CandidatoRepositoryOracle` (JDBC) a implementar:
  - `salvar`: transação; insere agregado + filhos.
  - `buscarPorId`: SELECT + filhos.
  - `buscarPorFiltros`: SQL dinâmico (case-insensitive).
  - `atualizar`: **substitutivo** (delete filhos + insert).
- Variáveis lidas em `DatabaseConfig`: `TC_DB_URL`, `TC_DB_USER`, `TC_DB_PASSWORD`.

> Quando quiser, posso entregar o **repository JDBC completo**.

---

## Boas Práticas & Convenções
- **Domínio puro** (sem dependências de framework).
- **DTO != Domínio** (mapeamento explícito).
- **Enums em UPPERCASE** (entradas devem respeitar).
- **Validação em duas camadas**: Bean Validation (borda) + Serviço de domínio (invariantes).
- **Idempotência** em atualizações: merge controlado no mapper.
- **Logs** informativos nos adapters (ex.: `salvo com id`).
- **Perfis bem definidos** (`mvp`/`oracle`).
- **Sem state global** (InMemory apenas para DEV/MVP).

---

## Troubleshooting (erros comuns)

- **`Cannot resolve symbol 'CandidatoMapper'`**  
  Falta o arquivo `CandidatoMapper.java` em `api/mapper`.

- **No CMD do Windows o `curl` quebra**  
  Use **PowerShell** (quebra com crase `` ` ``) ou mande **tudo em 1 linha** com aspas escapadas.  
  Melhor ainda: use **Thunder Client/Postman**.

- **`500` “Erro interno. Consulte os logs.” no POST**  
  Cheque validação de enums, datas `YYYY-MM-DD`, Bean Validation nos DTOs, stacktrace do backend.

- **Swagger não abre**  
  Tente `http://localhost:8080/swagger-ui/index.html`. Confira a dependência `springdoc`.

- **CORS no front (Angular)**  
  Adicione `@CrossOrigin(origins = "http://localhost:4200")` no controller (ou configure globalmente).

---

## Roadmap & Próximos Passos
1. **Front Angular** (feature `candidatos`): lista, filtros, form de cadastro/edição.
2. **Repository Oracle JDBC** (quando quiser integrar com o DB).
3. **Testes completos** (domínio, mapper, controller).
4. **Paginação/Ordenação reais** (no Oracle, `OFFSET/FETCH`).
5. **Autenticação JWT** e perfis de acesso.
6. **Dockerfile / docker-compose** (API + Oracle XE).
7. **Exportações** (CSV/Excel) e anexos (S3/Blob/MinIO).

---

## Apêndice – Exemplos práticos

### `curl` – **PowerShell** (quebra de linha com **crase**)
```powershell
curl -X POST http://localhost:8080/api/candidatos `
  -H "Content-Type: application/json" `
  -d '{
    "nomeCompleto": "Jon Doe",
    "contato": { "telefone": "81-98888-0000", "email": "jon.doe@talentcore.dev" },
    "endereco": { "cidade": "RECIFE", "estado": "PE", "pais": "BRASIL" }
  }'
```

### `curl` – **CMD** (tudo em **uma linha**)
```cmd
curl -X POST http://localhost:8080/api/candidatos -H "Content-Type: application/json" -d "{\"nomeCompleto\":\"Jon Doe\",\"contato\":{\"telefone\":\"81-98888-0000\",\"email\":\"jon.doe@talentcore.dev\"},\"endereco\":{\"cidade\":\"RECIFE\",\"estado\":\"PE\",\"pais\":\"BRASIL\"}}"
```

### Thunder Client/Postman
- Method: **POST**
- URL: `http://localhost:8080/api/candidatos`
- Body: **raw** → **JSON** (use os exemplos desta doc)

---

> © TalentCore. Mantido por Jonas M. R. da Silva.  
> Arquitetura Hexagonal • Java 17 • Spring Boot 3 • Oracle JDBC.
