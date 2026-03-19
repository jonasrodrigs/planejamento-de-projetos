# TalentCore — README (Frontend + Backend + Banco de Dados)

> **Objetivo**
>
> Este README documenta **tudo o que foi implementado** no projeto TalentCore: arquitetura de **Frontend (Angular)**, **Backend (Spring Boot)**, **Banco de Dados (Oracle)**, rotas de API, padrões de payload, fluxos de autenticação, passos de execução local, troubleshooting e roadmap. 
>
> O conteúdo reflete o estado atual do projeto após as últimas implementações: **telefone**, **fotoUrl**, **idiomas**, **habilidades técnicas**, **soft skills**, **experiências** (com tecnologias), **projetos** (com tecnologias), **disponibilidade**, **dados pessoais**, **links**, **localização** e **ocupação**.

---

## Índice

1. [Visão Geral](#visão-geral)
2. [Arquitetura de Pastas — Monorepo](#arquitetura-de-pastas--monorepo)
3. [Frontend (Angular)](#frontend-angular)
   - [Arquitetura de Pastas do Front](#arquitetura-de-pastas-do-front)
   - [Funcionalidades na página de Detalhe](#funcionalidades-na-página-de-detalhe)
   - [Estado, Signals e Helpers](#estado-signals-e-helpers)
   - [Execução](#execução-frontend)
4. [Backend (Spring Boot)](#backend-spring-boot)
   - [Arquitetura de Pastas do Back](#arquitetura-de-pastas-do-back)
   - [Port & Adapter (Oracle JDBC)](#port--adapter-oracle-jdbc)
   - [Autenticação (JWT)](#autenticação-jwt)
   - [Execução](#execução-backend)
5. [Banco de Dados (Oracle)](#banco-de-dados-oracle)
   - [Esquema / Tabelas](#esquema--tabelas)
   - [DDL Idempotente (Telefone)](#ddl-idempotente-telefone)
6. [API](#api)
   - [Autenticação](#autenticação)
   - [Candidatos — GET e PUT](#candidatos--get-e-put)
   - [Exemplos de Payload](#exemplos-de-payload)
7. [Troubleshooting](#troubleshooting)
8. [Boas Práticas e Convenções](#boas-práticas-e-convenções)
9. [Roadmap Curto](#roadmap-curto)
10. [Contribuição](#contribuição)

---

## Visão Geral

- **Frontend**: Angular (componentes standalone, **Signals** para estado reativo). Página de detalhe de talento estruturada em **cards** com renderização condicional (mostra apenas quando há dados).
- **Backend**: Spring Boot (Java 17) com arquitetura Port/Adapter. Persistência **Oracle JDBC** (sem JPA/Hibernate). Atualização de currículo com **transação**: UPDATE dos campos simples e **DELETE+INSERT** das coleções.
- **Banco de Dados**: Oracle. Tabelas normalizadas por agregado (CANDIDATO, IDIOMA, HABILIDADE, SOFT_SKILL, EXPERIENCIA, EXPERIENCIA_TECNOLOGIA, PROJETO, PROJETO_TECNOLOGIA). **Auto-detecção** de nome de colunas (ex.: `NOME` vs `IDIOMA`, `TECNOLOGIA` vs `NOME`) para suportar variações de schema.

---

## Arquitetura de Pastas — Monorepo

```
TalentCore/
├─ frontend/                      # Projeto Angular (standalone)
│  └─ src/app/
│     ├─ core/
│     │  └─ services/
│     │     └─ auth.service.ts
│     ├─ shared/
│     │  └─ layout/header-toolbar/
│     │     ├─ header-toolbar.component.ts
│     │     ├─ header-toolbar.component.html
│     │     └─ header-toolbar.component.scss
│     ├─ features/
│     │  └─ talents/
│     │     ├─ pages/
│     │     │  ├─ talent-list/
│     │     │  │  ├─ talent-list.component.ts
│     │     │  │  ├─ talent-list.component.html
│     │     │  │  └─ talent-list.component.scss
│     │     │  └─ talent-detail/
│     │     │     ├─ talent-detail.component.ts
│     │     │     ├─ talent-detail.component.html
│     │     │     └─ talent-detail.component.scss
│     │     └─ ... (guards, interceptors, rotas)
│     └─ ...
└─ backend/                       # Projeto Spring Boot (Java 17)
   └─ src/main/java/br/com/talentcore/talentos/
      ├─ TalentCoreApplication.java
      ├─ config/
      │  ├─ DatabaseConfig.java
      │  └─ SecurityConfig.java
      ├─ auth/
      │  ├─ AuthController.java
      │  ├─ AuthService.java
      │  ├─ JwtService.java
      │  ├─ JwtAuthFilter.java
      │  └─ dto/
      │     ├─ LoginRequest.java
      │     └─ LoginResponse.java
      ├─ domain/
      │  ├─ Candidato.java
      │  ├─ Endereco.java
      │  ├─ Disponibilidade.java
      │  ├─ Idioma.java
      │  ├─ Habilidade.java
      │  ├─ SoftSkill.java
      │  ├─ Experiencia.java
      │  ├─ Projeto.java
      │  └─ (enums: NivelIdioma, NivelConhecimento, TipoContratacao, ...)
      ├─ api/
      │  ├─ dto/
      │  │  ├─ CandidatoRequest.java
      │  │  └─ CandidatoResponse.java
      │  └─ mapper/
      │     └─ CandidatoMapper.java
      ├─ application/port/out/
      │  └─ CandidatoRepository.java
      └─ infrastructure/persistence/
         └─ CandidatoRepositoryOracle.java
```

---

## Frontend (Angular)

### Arquitetura de Pastas do Front
- `core/services/auth.service.ts`: login e gerenciamento de Bearer Token.
- `shared/layout/header-toolbar/*`: toolbar compartilhada.
- `features/talents/pages/talent-list/*`: listagem de talentos.
- `features/talents/pages/talent-detail/*`: detalhe do talento (TS/HTML/SCSS).

### Funcionalidades na página de Detalhe
Cards renderizados (condicionais):
1. **Resumo** (`resumoProfissional`).
2. **Links** (`linkedin`, `github`, `portfolio`).
3. **Contato** (`email`, `telefone`).
4. **Localização** (`bairro · cidade/UF`, `pais`).
5. **Ocupação** (`ocupacao`).
6. **Idiomas** (`idiomas: string[]`).
7. **Habilidades técnicas**:
   - `habilidades: string[]` **ou**
   - `habilidadesTecnicas: { nome, categoria?, nivel? }[]`.
8. **Soft skills** (`habilidadesComportamentais: { nome }[]`).
9. **Experiências** (`empresa`, `cargo`, `tipo`, `dataInicio`→`dataFim|Atual`, `tecnologias[]`, `realizacoes[]` opcional).
10. **Projetos** (`nome`, `descricao` opcional, `url` opcional, período, `tecnologias[]`).
11. **Tecnologias** agregadas (`tecnologias: string[]`).
12. **Dados pessoais** (`dataNascimento`, `nacionalidade`, `estadoCivil`).
13. **Disponibilidade** (`aceitaViagens`, `aceitaMudanca`, `horarios`, `pretensaoSalarial`).

### Estado, Signals e Helpers
- `loading`, `error`, `cand` (Signals).
- `hasPhoto`, `avatarInitials`, `hasResumo`, `enderecoCurto`, `hasDadosPessoais`, `hasDisponibilidade` (computed).
- `formatSimNao(v)` → "Sim"/"Não"/"—" para `'S'|'N'|true|false|"SIM"|"NAO"`.
- **Mapeamento defensivo** no TS: aceita ausência de campos sem quebrar a UI.

### Execução (Frontend)
```bash
cd frontend
npm install
npm start           # ou: ng serve
# Navegue em http://localhost:4200
```

---

## Backend (Spring Boot)

### Arquitetura de Pastas do Back
- `config/DatabaseConfig.java`: DataSource do Oracle.
- `config/SecurityConfig.java`: filtro JWT e autorização.
- `auth/*`: controller/serviço/filtro para login e geração/validação do token.
- `domain/*`: modelos de domínio.
- `api/dto/*`: DTOs de request/response.
- `api/mapper/CandidatoMapper.java`: transformação domínio ↔ DTO.
- `application/port/out/CandidatoRepository.java`: contrato (Port).
- `infrastructure/persistence/CandidatoRepositoryOracle.java`: JDBC Adapter (Oracle).

### Port & Adapter (Oracle JDBC)
**Coleções implementadas** (GET + PUT `atualizarCurriculo` com transação):
- **Idiomas** (`IDIOMA`) com auto-detecção da coluna de nome (`NOME` **ou** `IDIOMA`).
- **Habilidades técnicas** (`HABILIDADE`).
- **Soft skills** (`SOFT_SKILL`).
- **Experiências** (`EXPERIENCIA`) + **Tecnologias** (`EXPERIENCIA_TECNOLOGIA`, nome: `TECNOLOGIA` **ou** `NOME`).
- **Projetos** (`PROJETO`) + **Tecnologias** (`PROJETO_TECNOLOGIA`, nome: `TECNOLOGIA` **ou** `NOME`).

**Campos principais suportados em `CANDIDATO`:** telefone, país, fotoUrl, ocupação, resumo, links, dados pessoais, disponibilidade (S/N), endereço (bairro/cidade/UF/pais), pretensão.

### Autenticação (JWT)
- `POST /auth/login` → `{ token }`.
- Enviar `Authorization: Bearer <token>` em rotas protegidas.

### Execução (Backend)
```bash
cd backend
# Configure o acesso ao Oracle em application.properties (ou equivalente)
mvn spring-boot:run
# ou
mvn clean package && java -jar target/*.jar
```
**Config. típica**
```properties
spring.datasource.url=jdbc:oracle:thin:@//host:1521/ORCLCDB.localdomain
spring.datasource.username=TALENTCORE
spring.datasource.password=******
spring.datasource.driver-class-name=oracle.jdbc.OracleDriver
server.port=8080
jwt.secret=troque-este-segredo
jwt.expiration=3600000
```

---

## Banco de Dados (Oracle)

### Esquema / Tabelas

#### `CANDIDATO`
- `ID (PK)`, `NOME_COMPLETO`, `DATA_NASCIMENTO (DATE)`, `EMAIL`, `TELEFONE`, `CIDADE`, `UF`, `BAIRRO`, `PAIS`,
  `OCUPACAO`, `RESUMO_PROFISSIONAL (CLOB)`, `LINKEDIN`, `GITHUB`, `PORTFOLIO`, `FOTO_URL`,
  `NACIONALIDADE`, `ESTADO_CIVIL`, `PRETENSAO_SALARIAL`, `HORARIOS`, `ACEITA_VIAGENS (S/N)`, `ACEITA_MUDANCA (S/N)`.

#### `IDIOMA`
- `ID (PK)`, `CANDIDATO_ID (FK)`, **`NOME` _ou_ `IDIOMA`** (auto-detect), `NIVEL` (`BASICO|INTERMEDIARIO|AVANCADO|FLUENTE|NATIVO`).

#### `HABILIDADE`
- `ID (PK)`, `CANDIDATO_ID (FK)`, `NOME`, `CATEGORIA`, `NIVEL` (`BASICO|INTERMEDIARIO|AVANCADO|EXPERT`).

#### `SOFT_SKILL`
- `ID (PK)`, `CANDIDATO_ID (FK)`, `NOME`.

#### `EXPERIENCIA`
- `ID (PK)`, `CANDIDATO_ID (FK)`, `EMPRESA`, `CARGO`, `TIPO` (`CLT|PJ|ESTAGIO|FREELANCE`), `DATA_INICIO`, `DATA_FIM`.

#### `EXPERIENCIA_TECNOLOGIA`
- `ID (PK)`, `EXPERIENCIA_ID (FK)`, **`TECNOLOGIA` _ou_ `NOME`** (auto-detect).

#### `PROJETO`
- `ID (PK)`, `CANDIDATO_ID (FK)`, `NOME`, `DESCRICAO (opcional)`, `URL (opcional)`, `DATA_INICIO`, `DATA_FIM`.

#### `PROJETO_TECNOLOGIA`
- `ID (PK)`, `PROJETO_ID (FK)`, **`TECNOLOGIA` _ou_ `NOME`** (auto-detect).

### DDL Idempotente (Telefone)
Caso ainda não exista a coluna `TELEFONE` em `CANDIDATO`:
```sql
DECLARE
  v_cnt INTEGER;
BEGIN
  SELECT COUNT(*) INTO v_cnt
  FROM USER_TAB_COLS
  WHERE UPPER(TABLE_NAME) = 'CANDIDATO' AND UPPER(COLUMN_NAME) = 'TELEFONE';
  IF v_cnt = 0 THEN
    EXECUTE IMMEDIATE 'ALTER TABLE CANDIDATO ADD (TELEFONE VARCHAR2(20))';
  END IF;
END;
/
```

---

## API

### Autenticação
```
POST /auth/login
{
  "email": "admin@talentcore.com",
  "password": "123456"
}
```
**Resposta**: `{ "token": "..." }`

Usar o token nas requisitações seguintes:
```
Authorization: Bearer <token>
```

### Candidatos — GET e PUT
- **GET** `/api/candidatos/{id}` → retorna um `CandidatoResponse` com todos os campos suportados pelo front.
- **PUT** `/api/candidatos/{id}` → atualiza campos simples e **sincroniza** coleções (`idiomas`, `habilidadesTecnicas`, `habilidadesComportamentais`, `experiencias`, `projetos`).

### Exemplos de Payload

**PUT mínimo + experiências e projetos**
```json
{
  "nomeCompleto": "Carla Fullstack",
  "contato": { "email": "carla.fullstack@talentcore.dev", "telefone": "+55 81 98888-0000" },
  "endereco": { "cidade": "OSASCO", "estado": "SP", "pais": "BRASIL" },

  "fotoUrl": "https://i.pravatar.cc/300?img=5",

  "idiomas": [
    { "idioma": "Inglês",  "nivel": "AVANCADO" },
    { "idioma": "Espanhol","nivel": "INTERMEDIARIO" }
  ],

  "habilidadesTecnicas": [
    { "nome": "Angular 17", "categoria": "Frontend", "nivel": "AVANCADO" },
    { "nome": "Java 17",    "categoria": "Backend",  "nivel": "INTERMEDIARIO" }
  ],

  "habilidadesComportamentais": [
    { "nome": "Comunicação" },
    { "nome": "Trabalho em equipe" }
  ],

  "experiencias": [
    {
      "empresa": "Tech Co",
      "cargo": "Fullstack Dev",
      "tipo": "CLT",
      "dataInicio": "2022-01-01",
      "dataFim": "2024-12-01",
      "tecnologias": ["Angular 17", "Java 17", "Spring Boot", "Oracle"],
      "realizacoes": ["Reduziu tempo de build", "Implementou design system"]
    }
  ],

  "projetos": [
    {
      "nome": "Portal Corporativo",
      "descricao": "Portal de autosserviço com autenticação e dashboard.",
      "url": "https://demo.carla.dev/portal",
      "dataInicio": "2023-01-01",
      "dataFim": "2023-10-01",
      "tecnologias": ["Angular 17", "RxJS", "SCSS", "JWT"]
    }
  ],

  "ocupacao": "Desenvolvedora Fullstack",
  "resumoProfissional": "Desenvolvedora Fullstack com experiência em Angular e Java...",
  "linkedin": "https://linkedin.com/in/carla-fullstack",
  "github": "https://github.com/carla-fullstack",
  "portfolio": "https://carla.dev",
  "nacionalidade": "Brasileira",
  "estadoCivil": "Solteira",
  "aceitaViagens": true,
  "aceitaMudanca": false,
  "horarios": "Comercial",
  "pretensaoSalarial": "R$ 6.000 – 8.000 CLT"
}
```

---

## Troubleshooting

### Erro `ORA-00904: invalid identifier`
- Indica divergência de nome de coluna no schema.
- O adapter usa **auto-detecção** para `NOME|IDIOMA` (tabela `IDIOMA`) e `TECNOLOGIA|NOME` (tabelas de tecnologias). Se persistir:
  1. Liste colunas: `SELECT COLUMN_NAME FROM USER_TAB_COLS WHERE UPPER(TABLE_NAME) = 'NOME_DA_TABELA';`
  2. Ajuste os candidatos passados a `resolveNomeCol(...)` se necessário.

### Telefone não persiste
- Verifique se a coluna `TELEFONE` existe em `CANDIDATO`. Use a [DDL idempotente](#ddl-idempotente-telefone).

### PUT aceito, mas GET sem dados da coleção
- Lembre-se: `atualizarCurriculo` faz **DELETE+INSERT** das coleções. Confirme se o body do PUT contém as listas.
- Confira permissões do usuário no schema Oracle.

### Front não mostra card
- Faça **hard reload** (Ctrl + Shift + R).
- Confirme que o JSON do GET possui a propriedade esperada (`experiencias`, `projetos`, etc.).

---

## Boas Práticas e Convenções

- **Enums em UPPERCASE**:
  - Idiomas: `BASICO|INTERMEDIARIO|AVANCADO|FLUENTE|NATIVO`.
  - Habilidades: `BASICO|INTERMEDIARIO|AVANCADO|EXPERT`.
  - Experiência (tipo): `CLT|PJ|ESTAGIO|FREELANCE`.
- **Datas**: ISO `YYYY-MM-DD`.
- **Transacional**: manter coleções via `DELETE + INSERT` dentro de `atualizarCurriculo`.
- **UI**: cards condicionais; não renderizar campos vazios; usar chips para listas curtas.
- **Segurança**: manter `jwt.secret` seguro e rotacionável.

---

## Roadmap Curto

- Formatar datas de experiências/projetos (`dd/MM/yyyy`) diretamente no template.
- Filtros na lista por **tecnologia/nível/cidade/idioma**.
- Upload de CV/foto com armazenamento e URL assinada.
- Paginação/ordenação nos endpoints de listagem.
- Testes automatizados (unit/integration) e pipeline de CI.

---

## Contribuição

1. Crie uma branch por feature: `feature/<nome-claro>`.
2. Commits semânticos: `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`.
3. PR com descrição do que mudou. **Evite PRs gigantes** — priorize alterações **um arquivo por vez**.

---

> Dúvidas ou melhorias? Abra uma issue ou descreva o que precisa ser ajustado. Este README foi escrito para ser **copiado/colado** no repositório e servir de **guia único** para subir, testar e evoluir o TalentCore.
