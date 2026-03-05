# 🧭 TalentCore — Documentação Técnica

> **Resumo:** Plataforma tipo “LinkedIn de currículos”, **sem feed social**.  
> **Perfis:**  
> • **Candidato** – cadastra e mantém currículo completo.  
> • **Recrutador** – **busca** por filtros e **contata**.

---

## 1) Visão Geral do Produto

- **Objetivo:** Ajudar recrutadores a encontrar profissionais com base em dados de currículo (experiências, formações, habilidades, idiomas etc.).  
- **Escopo (MVP):**
  - Cadastro/validação de **Candidato** (agregado raiz).  
  - Persistência de **dados do candidato** (tabela `CANDIDATO`).  
  - **Busca por ID** e **estrutura** para futura busca por filtros.  
- **Foco:** simplicidade, clareza e arquitetura limpa.

---

## 2) Arquitetura — Hexagonal (Ports & Adapters)

```
              +------------------------+
              |     Infrastructure     |  ← JDBC/Oracle, HTTP (adapters)
              +-----------+------------+
                          ▲
                          | (adapters)
                          |
              +-----------+------------+
              |        Application     |  ← Use cases & Ports (in/out)
              +-----------+------------+
                          ▲
                          | (interfaces)
                          |
                  +-------+-------+
                  |     Domain    |  ← Entidades, VOs, Enums, Regras
                  +---------------+
```

**Princípios-chave:**
- **Domínio** não depende de tecnologia.
- **Use cases** orquestram e conversam via **ports**.
- **Infra** implementa detalhes (JDBC/HTTP).

---

## 3) Estrutura de Pastas

**Base:** `src/main/java`

```
br/com/talentcore/talentos
├─ domain
│  ├─ (entidades, VOs, enums)
│  └─ service
├─ application
│  └─ port
│     ├─ in
│     └─ out
├─ infrastructure
│  ├─ http
│  └─ persistence
└─ config
```

> **Nota:** O **package** inicia em `br.com…` (não use `main.java` no package).

---

## 4) Modelo de Domínio (Core)

### 4.1 Entidades (em `br.com.talentcore.talentos.domain`)
- **Candidato** *(agregado raiz)*
- **Formacao**
- **Experiencia**
- **Habilidade** (técnica)
- **SoftSkill**
- **Certificacao**
- **Curso**
- **Idioma**
- **Projeto**
- **Conquista**
- **ReferenciaProfissional**
- **Anexo**

### 4.2 Value Objects (VOs)
- **Contato** – `telefone`, `email`
- **Endereco** – `logradouro`, `numero`, `complemento`, `bairro`, `cidade`, `estado`, `pais`, `cep`
- **Disponibilidade** – `aceitaViagens`, `aceitaMudanca`, `horarios`

> ✅ **Getters/Setters** foram gerados (especialmente para `Endereco` e `Disponibilidade`), permitindo o uso no JDBC (`getLogradouro()`, `getHorarios()`, etc.).

### 4.3 Enums
- `NivelFormacao` – `TECNICO`, `GRADUACAO`, `POS`, `MBA`, `MESTRADO`, `DOUTORADO`  
- `TipoContratacao` – `CLT`, `PJ`, `ESTAGIO`, `FREELANCE`  
- `NivelConhecimento` – `BASICO`, `INTERMEDIARIO`, `AVANCADO`, `EXPERT`  
- `NivelIdioma` – `BASICO`, `INTERMEDIARIO`, `AVANCADO`, `FLUENTE`, `NATIVO`  
- `TipoAnexo` – `CURRICULO`, `PORTFOLIO`, `CARTA`

---

## 5) Regras de Negócio (Domain Service)

**Classe:** `br.com.talentcore.talentos.domain.service.CandidatoService`

- **Validações:**
  - Nome do candidato é **obrigatório**.  
  - **Listas não nulas** (experiências, cursos, etc.).  
  - **Coerência de datas**:  
    - `Experiencia`: `dataFim` ≥ `dataInicio`  
    - `Formacao`: `dataFim` ≥ `dataInicio`  
    - `Certificacao`: `dataExpiracao` ≥ `dataObtencao`  
- **Normalização simples:** `trim()` em tecnologias/realizações.

> **Dica:** se pretender **adicionar** itens às listas, use listas **mutáveis** ao garantir não-nulo.

---

## 6) Application Layer — Use Cases & Ports

### 6.1 Ports (interfaces)
- **Entrada (in):**
  - `CadastrarCandidatoUseCase` → `String executar(Candidato candidato)`
  - `BuscarCandidatoUseCase` → `List<Candidato> executar(Filtros filtros)`

- **Saída (out):**
  - `CandidatoRepository` →  
    `String salvar(Candidato)`,  
    `Optional<Candidato> buscarPorId(String id)`,  
    `List<Candidato> buscarPorFiltros(...filtros)`

### 6.2 Implementações de Use Case (em `application`)
- **`CadastrarCandidatoService`**
  - Gera `UUID` se `id` vier vazio
  - Valida com `CandidatoService`
  - Persiste via `CandidatoRepository.salvar`

- **`BuscarCandidatoService`**
  - Orquestra a chamada a `repo.buscarPorFiltros(...)` (MVP, pronto para evoluir)

---

## 7) Config & Infrastructure

### 7.1 `config/DatabaseConfig.java`
- Fornece `getConnection()` (JDBC/Oracle) via `DriverManager`.
- Parametrização `URL/USER/PASS` (ajustar para seu ambiente).

### 7.2 `infrastructure/persistence/CandidatoRepositoryOracle.java`
- Implementa `CandidatoRepository` com **JDBC puro**.
- **Inclui:**
  - `salvar(Candidato)` → `INSERT` em `CANDIDATO`
  - `buscarPorId(String)` → `SELECT` em `CANDIDATO` (mapeia também `Contato`, `Endereco`, `Disponibilidade`)
  - `buscarPorFiltros(...)` → **a implementar** (lança `UnsupportedOperationException` no MVP)
- **Cuidados tomados:**
  - **Null-safety** nos VOs (ternário com `d != null ? ... : null`)
  - **Conversão de data** (`LocalDate` ↔ `java.sql.Date`)
  - **Text blocks** no SQL (se language level < 15, usar `String` concatenada)

> **Nota:** neste MVP não persistimos/lemos as coleções (experiências, formações etc.). Isso vem em seguida, com **transação**.

---

## 8) Fluxos (alto nível)

### **Cadastro de candidato**
1. Controller / UI chama `CadastrarCandidatoUseCase.executar(candidato)`  
2. Gera `UUID` se necessário  
3. Validações de domínio  
4. `CandidatoRepository.salvar` (INSERT `CANDIDATO`)  
5. Retorna `id`

### **Busca por ID**
1. `CandidatoRepository.buscarPorId(id)`  
2. Faz `SELECT` em `CANDIDATO`  
3. Reconstrói `Candidato` com VOs  
4. Retorna `Optional<Candidato>`

---

## 9) Convenções & Boas Práticas

- **Packages**: sempre **minúsculos**, sem espaços/hífens.
- **Domínio limpo**: sem imports de JDBC/HTTP/JSON no `domain`.
- **Use cases**: orquestram regras e portas; sem dependências de infra concreta.
- **Infra**: detalhe técnico (JDBC/HTTP), sem lógica de domínio.
- **DTOs** (se necessário): na borda (controller), mapeando para o domínio.

---

## 10) Próximos Passos

1) **Implementar `buscarPorFiltros(...)`** (Oracle SQL):
   - Por **tecnologia** (em experiência/projeto), **nível** de habilidade, **cidade/estado**, **idioma/nivelIdioma**.  
   - Estratégias: `JOIN`/`EXISTS` com índices (`*_CANDIDATO_ID`).

2) **Persistir coleções** do candidato (experiências, formações, etc.) com **transação**:
   - `INSERT` em `CANDIDATO`  
   - `INSERT` nas tabelas filhas (com `CANDIDATO_ID`)  
   - `COMMIT`/`ROLLBACK` na falha.

3) **Teste de integração** simples (sem framework):
   - Montar `Candidato` → `CadastrarCandidatoService` → `buscarPorId`.

4) (Opcional) **Controller HTTP** minimalista (sem framework) para testes com `curl` / seu front em Angular.

---

## 11) Checklist do que já foi feito ✅

- [x] **Estrutura de pastas** (domain, application, infrastructure, config)  
- [x] **Domínio completo** (entidades, VOs, enums) + **getters/setters**  
- [x] **CandidatoService** (validações de negócio)  
- [x] **Ports** (`CadastrarCandidatoUseCase`, `BuscarCandidatoUseCase`, `CandidatoRepository`)  
- [x] **Implementações** (`CadastrarCandidatoService`, `BuscarCandidatoService`)  
- [x] **DatabaseConfig** (JDBC/Oracle)  
- [x] **CandidatoRepositoryOracle** (INSERT + SELECT por ID)

**Pendências principais:**
- [ ] `buscarPorFiltros(...)` no repositório  
- [ ] Persistir/consultar **coleções** (experiências, formações etc.)  
- [ ] Controller HTTP (opcional, para POC)

---

## 12) Observações de Ambiente

- **JDK recomendado:** 17+  
  - Se usar `String.isBlank()` e *text blocks* (`""" ... """`), precisa de language level adequado (11+ e 15+).  
  - Alternativas com `trim().isEmpty()` e `String` concatenada estão indicadas no código.

- **Driver Oracle:** inclua `ojdbc8.jar` no classpath (ou dependência no Maven/Gradle).

