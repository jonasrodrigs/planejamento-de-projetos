
# TalentCore — Recap Técnico (Frontend + Backend + Banco)

> **Propósito**: este README consolida *tudo* que foi feito até aqui para que você possa
> 1) versionar no GitHub, 2) colar em outro chat/assistente e seguir o desenvolvimento,
> 3) relembrar a arquitetura e os passos de execução rapidamente.
>
> **Atualizado em**: 2026-03-16 21:22:39

---

## 📦 Visão Geral do Projeto

**Stack**
- **Frontend**: Angular 17 (standalone, Signals), TypeScript, HTML/SCSS
- **Backend**: Spring Boot 3.3, Java 17, Spring Security (JWT), JDBC (Oracle)
- **Banco**: Oracle (PDB `XEPDB1`, schema `TALENTCORE`)

**Objetivo funcional**
- Aplicação de **banco de talentos** com **lista**, **dashboard** e **detalhe em formato de currículo**, com autenticação via **JWT**.

---

## 🗂️ Arquitetura de Pastas (alto nível)

```
TalentCore/
├─ frontend/                      # Projeto Angular
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
└─ backend/                       # Projeto Spring Boot
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
      │  └─ Disponibilidade.java
      ├─ api/
      │  ├─ dto/CandidatoResponse.java
      │  └─ mapper/CandidatoMapper.java
      ├─ application/port/out/
      │  └─ CandidatoRepository.java
      └─ infrastructure/persistence/
         └─ CandidatoRepositoryOracle.java
```

---

## ✅ O que foi feito — Frontend

### 1) **Detalhe do Talento (formato de currículo)**
Arquivos:
- `talent-detail.component.ts`
- `talent-detail.component.html`
- `talent-detail.component.scss`

**Principais recursos**
- **Foto opcional** (`fotoUrl`) + **avatar com iniciais** (fallback)
- **Resumo profissional** (`resumoProfissional`) com `white-space: pre-line`
- **Links** (LinkedIn, GitHub, Portfólio) como **chips clicáveis**
- **Endereço** compacto: `Bairro · Cidade/UF` (pipe `titlecase` no template)
- **Dados pessoais**: `dataNascimento | date: 'dd/MM/yyyy'`, `nacionalidade`, `estadoCivil`
- **Disponibilidade**: `aceitaViagens`/`aceitaMudanca` (render `Sim/Não`), `horarios`, `pretensaoSalarial`
- **Experiências** e **Projetos** (cards *full width*, bullets e chips de tecnologias)
- **Responsivo** e **print‑friendly** (sem botões na impressão, contraste ajustado)

### 2) **Lista de Talentos**
Arquivos:
- `talent-list.component.ts` (com `trackById` e `HttpParams` em string)
- `talent-list.component.html` (colunas: Nome | E-mail | Cidade/UF | Ocupação | Idiomas | Ações)

**Detalhes**
- Leitura via `GET /api/candidatos?page=0&size=200[&q=]`
- Exibe **Ocupação** (campo universal do currículo)
- Idiomas em chips (máx. 3 + contador `+N`)

### 3) **Header / Busca**
Arquivos:
- `header-toolbar.component.ts` (ajuste de visibilidade da barra de busca)

**Regras**
- Barra de busca **apenas** na rota **exata** `/app/talents` (não exibe em `/app/talents/:id`)

### 4) **Autenticação no Front**
- Login Angular + `AuthService` com armazenamento de token
- Interceptor para **401 → /login** e **403 → /dashboard**
- Auto-logout por expiração do `exp` (JWT)

---

## ✅ O que foi feito — Backend

### 1) **Auth (login) simplificado**
Arquivos:
- `AuthController.java` (`POST /auth/login`)
- `AuthService.java` (usa `AuthenticationManager` → gera JWT via `JwtService`)
- `LoginRequest` com `@JsonAlias` para aceitar `email/username` e `password/senha`
- `SecurityConfig.java` (perfil atual com **InMemoryUserDetailsManager** para DEV, usuários: `admin@talentcore.dev/.com` senhas `123456`, `BCryptPasswordEncoder`)

### 2) **Domínio**
- `Endereco.java` — inclui **bairro** e helper `toEnderecoCurto()`
- `Disponibilidade.java` — usa `Boolean` (nulo = desconhecido) + helpers `getAceita*SN()` e setters from `S/N`
- `Candidato.java` — adicionados: `ocupacao`, `resumoProfissional`, `linkedin`, `github`, `portfolio`, dados pessoais e disponibilidade

### 3) **DTO + Mapper**
- `CandidatoResponse.java` — **flat** para consumo Angular: básicos + endereço + currículo + disponibilidade + listas
- `CandidatoMapper.toResponse()` — mapeia domínio → DTO (datas ISO `yyyy-MM-dd`, endereço flat, disponibilidade Booleans)

### 4) **Repository (Oracle/JDBC)**
Arquivo:
- `CandidatoRepositoryOracle.java`

**Mudanças**
- **SELECT (lista e detalhe)** passaram a retornar também:
  - `OCUPACAO`, `RESUMO_PROFISSIONAL`, `BAIRRO`, `LINKEDIN`, `GITHUB`, `PORTFOLIO`
- `mapRow(rs)` agora popula esses campos no `Candidato` (inclui `bairro` em `Endereco`)
- `salvar/atualizar` **mantidos mínimos** (não gravam currículos ainda — leitura primeiro)

### 5) **Banco (Oracle)**

**ALTER TABLE idempotente**:
- Colunas criadas/confirmadas na `CANDIDATO`:
  - `OCUPACAO VARCHAR2(120)`
  - `RESUMO_PROFISSIONAL CLOB`
  - `BAIRRO VARCHAR2(100)`
  - `LINKEDIN VARCHAR2(300)`
  - `GITHUB VARCHAR2(300)`
  - `PORTFOLIO VARCHAR2(300)`
  - (opcionais previstos) `NACIONALIDADE`, `ESTADO_CIVIL`, `PRETENSAO_SALARIAL`, `ACEITA_VIAGENS CHAR(1)`, `ACEITA_MUDANCA CHAR(1)`, `HORARIOS`

**Seed por ID (MERGE)** — exemplos aplicados:
- `Carla Fullstack (ID=f374c01c-...)`: ocupação, resumo, bairro, links, disponibilidade
- `Bruno Backend (ID=eeb6abde-...)`: ocupação, resumo, bairro, links, disponibilidade
- `Alice Front (ID=a31c1f97-...)`: ocupação, resumo, bairro, links, disponibilidade

**Autenticação (DEV)**
- Tabelas padrão (se optar por JDBC em vez de in‑memory):
  - `USERS(username, password, enabled)`
  - `AUTHORITIES(username, authority)`
- Usuário DEV: `admin@talentcore.com` com **BCrypt(123456)**

---

## ▶️ Como rodar (DEV)

### Backend (profile oracle)
- **Variáveis** (Run Configuration):
  ```
  TC_DB_URL=jdbc:oracle:thin:@//<HOST>:<PORT>/<PDB>   # ex.: //192.168.101.10:1521/XEPDB1
  TC_DB_USER=TALENTCORE
  TC_DB_PASSWORD=********
  JWT_SECRET=<uma chave >= 32 chars>
  ```
- **Profile**: `oracle`
- **Run**: IDE (▶️) ou `mvn spring-boot:run -Dspring-boot.run.profiles=oracle`

### Login (Thunder/Postman)
```http
POST http://localhost:8080/auth/login
Content-Type: application/json

{
  "email": "admin@talentcore.com",
  "password": "123456"
}
```
→ copie o `token` e use como **Bearer** nos GETs.

### Endpoints de teste
```http
GET http://localhost:8080/api/candidatos               # lista
GET http://localhost:8080/api/candidatos/{id}          # detalhe
```

### Frontend (Angular)
- `npm install`
- `ng serve` (ou `ng serve --proxy-config proxy.conf.json` para proxy do `/api`)
- Acessar: `http://localhost:4200`

---

## 🧭 Dicas rápidas (diagnóstico)
- **403 no Thunder** → faltou **Bearer**; na aba **Auth** selecione `Bearer` e cole **apenas** o valor do token (sem `{ "token": ... }`).
- **Detalhe ok, lista nula** → ver SELECT da lista no repository (agora já traz campos de currículo).
- **Front sem atualizar** → **Ctrl+Shift+R** (hard reload) no navegador.

---

## 🗺️ Próximos passos sugeridos (incremental, simples)
1) **Persistir currículo no UPDATE/INSERT** (Oracle): gravar `OCUPACAO`, `RESUMO_PROFISSIONAL`, `BAIRRO`, `LINKEDIN/GITHUB/PORTFOLIO`, disponibilidade.
2) **Idiomas no Dashboard**:
   - Opção A: `IDIOMAS_TXT` em `CANDIDATO` + `split(',')` no mapper (rápido)
   - Opção B: tabela `CANDIDATO_IDIOMA` (relacional)
3) **Foto**: `FOTO_URL VARCHAR2(500)` + repository (front já tem fallback).
4) **Form de edição** no front (Reactive Forms) + `PUT /api/candidatos/{id}`.

---

## 🧾 Changelog (sessão atual)
- Front **Detalhe**: Foto/Avatar, Resumo, Links, Endereço, Dados pessoais, Disponibilidade, Experiências/Projetos, SCSS responsivo/print.
- Front **Lista**: `trackById`, coluna **Ocupação**, chips de Idiomas.
- Header: busca **só** em `/app/talents`.
- Back **Domínio**: `Candidato`, `Endereco` (bairro), `Disponibilidade` (Boolean + helpers).
- Back **DTO/Mapper**: `CandidatoResponse` flat; `CandidatoMapper.toResponse()` atualizado.
- Back **Repository**: SELECTs com `OCUPACAO`, `RESUMO_PROFISSIONAL`, `BAIRRO`, `LINKEDIN`, `GITHUB`, `PORTFOLIO`.
- Banco: `ALTER TABLE` e `MERGE` por ID (Carla/Bruno/Alice).
- Auth: `SecurityConfig` com **InMemory** para DEV; login **POST /auth/login** + JWT; uso correto do Bearer no Thunder.

---

## ✍️ Créditos & Notas
- Este README foi gerado a partir do histórico de implementação da sessão, com foco em
  **clareza** e **ação**. É seguro para colar em outro chat e continuar o desenvolvimento.

