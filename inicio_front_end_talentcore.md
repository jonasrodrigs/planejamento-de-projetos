
# TalentCore

Backend em **Java 17 + Spring Boot 3** (Arquitetura Hexagonal) e Frontend em **Angular 17 (standalone)**.  
Autenticação via **JWT**, rotas protegidas com **guards**, **interceptor** para 401/403, proxy de desenvolvimento e separação clara entre área pública e autenticada (`/app/**`).

---

## Sumário
- [Visão Geral](#visão-geral)
- [Arquitetura](#arquitetura)
- [Stack Tecnológica](#stack-tecnológica)
- [Estrutura de Pastas](#estrutura-de-pastas)
- [Ambientes & Variáveis](#ambientes--variáveis)
- [Como Rodar (DEV)](#como-rodar-dev)
- [Fluxo de Autenticação (JWT)](#fluxo-de-autenticação-jwt)
- [Rotas do Front](#rotas-do-front)
- [Endpoints Principais (Back)](#endpoints-principais-back)
- [Boas Práticas & Padrões](#boas-práticas--padrões)
- [Checklist de Status](#checklist-de-status)
- [Próximos Passos Sugeridos](#próximos-passos-sugeridos)
- [Troubleshooting Rápido](#troubleshooting-rápido)
- [Licença](#licença)

---

## Visão Geral
O **TalentCore** é um sistema de **gestão de talentos** com backend desacoplado (Hexagonal) e frontend moderno (Angular).  
Foco em **simplicidade**, **segurança** (JWT) e **evolutividade** (ports/adapters, DTOs e mapeadores no front).

---

## Arquitetura
**Hexagonal / Ports & Adapters (Back):**
- **domain** (puro): entidades, VOs, enums e regras
- **application**: casos de uso (ports `in/out`) e services
- **infrastructure**: adapters de entrada (HTTP) e saída (persistência)
- **auth** e **config** isolados

**Angular (Front):**
- **core**: services, interceptors, guards, environments
- **features**: domínios (auth, dashboard, talents)
- **shared**: layout e componentes comuns
- Roteamento **standalone** com **`canMatch`** para área autenticada `(/app/**)`

---

## Stack Tecnológica
**Backend**
- Java 17, Spring Boot 3.3.x
- Spring Web, Security, Validation, Actuator
- JJWT (`io.jsonwebtoken`)
- JDBC (Oracle em perfil futuro) e Repositório **InMemory** para MVP
- Swagger/OpenAPI (springdoc)

**Frontend**
- Angular 17 (standalone)
- Reactive Forms, Router, HttpClient
- Interceptors (JWT), Guards (`canMatch`)
- Proxy de desenvolvimento

---

## Estrutura de Pastas
### Backend (resumo)
```
src/main/java/br/com/talentcore/talentos/
├─ TalentCoreApplication.java
├─ auth/ (AuthController, AuthService, JwtAuthFilter, JwtService, dto/)
├─ config/ (SecurityConfig, BeanConfig, DatabaseConfig, OpenApiConfig)
├─ domain/ (entidades, VOs, enums, domain.service/)
├─ application/
│  ├─ port/in (UseCases)
│  ├─ port/out (Ports de saída)
│  └─ services (implementações dos UseCases)
└─ infrastructure/
   ├─ http/ (controllers, handlers)
   └─ persistence/
      ├─ mvp/ (InMemory)
      └─ oracle/ (adapter Oracle – futuro)
```

### Frontend (resumo)
```
src/app/
├─ app.routes.ts  (rotas públicas e /app/** autenticadas)
├─ app.config.ts  (HttpClient+Interceptor, Router, Title)
├─ core/
│  ├─ services/ (auth.service.ts)
│  ├─ interceptors/ (auth.interceptor.ts)
│  └─ guards/ (auth.guard.ts)
├─ features/
│  ├─ auth/ (login page)
│  ├─ dashboard/ (dashboard-home.component)
│  └─ talents/ (lista/rotas)
└─ shared/
   └─ layout/
      ├─ header-toolbar/ (logout, saudação, links)
      └─ main-layout/
```

---

## Ambientes & Variáveis
**Frontend**
- `src/environments/environment.ts`
  ```ts
  export const environment = { production: false, apiUrl: '' };
  ```
- `proxy.conf.json`
  ```json
  {
    "/api":  { "target": "http://localhost:8080", "secure": false, "changeOrigin": true },
    "/auth": { "target": "http://localhost:8080", "secure": false, "changeOrigin": true }
  }
  ```

**Backend**
- `application.yml` (perfil `mvp` por padrão, sem DB)
- JWT:
  ```yaml
  security:
    jwt:
      secret: ${JWT_SECRET:dev-secret-please-change-32bytes-or-more-123456}
      exp-minutes: ${JWT_EXP_MINUTES:120}
  ```
- (Futuro) Oracle via variáveis:
  ```
  TC_DB_URL, TC_DB_USER, TC_DB_PASSWORD
  ```

---

## Como Rodar (DEV)
### Backend
```bash
mvn spring-boot:run
# ou
./mvnw spring-boot:run
```
Acessos úteis:
- API: `http://localhost:8080`
- Swagger: `http://localhost:8080/swagger-ui/index.html`

**Credenciais DEV (in-memory SecurityConfig)**
- `admin@talentcore.dev / 123456`
- `user@talentcore.dev  / 123456`

### Frontend
```bash
npm install
ng serve --proxy-config proxy.conf.json
# abre em http://localhost:4200
```

---

## Fluxo de Autenticação (JWT)
1. **Login**: `POST /auth/login` (email/senha) → retorna `{ token }`
2. Front salva **token** em `localStorage` (ou `sessionStorage` conforme “Lembrar-me”)
3. **Interceptor** adiciona `Authorization: Bearer ...` para `/api/**`
4. **Guard (`canMatch`)** protege `/app/**`
5. **Auto-logout** no front quando `exp` do JWT é alcançado
6. **Tratamento 401/403** no interceptor:
   - 401 → `/login?reason=expired`
   - 403 → `/dashboard?reason=forbidden`

---

## Rotas do Front
- **Públicas**:
  - `/login` (suporta `?returnUrl=/app/...` e `?reason=expired|forbidden|logout`)
- **Autenticadas (área segura)**:
  - `/app/dashboard`
  - `/app/talents`
  - (fallback interno: `/app` → `/app/dashboard`)

---

## Endpoints Principais (Back)
- `POST /auth/login` → Autenticação, retorna `{ token }`
- `GET /api/candidatos` → Filtro/paginação (MVP em memória)
- `POST /api/candidatos` → Cadastro
- `PUT /api/candidatos/{id}` → Atualização
- `GET /api/candidatos/{id}` → Busca por ID
> Todas as rotas `/api/**` exigem **JWT** válido.

---

## Boas Práticas & Padrões
- **Hexagonal**: ports (`application/port`), adapters (`infrastructure`), domínio puro
- **Use Cases como interfaces** (`port/in`) e implementações em `application/services`
- **DTOs/Mapper** explicitando contrato entre front/back
- **Angular standalone**: `canMatch` em rotas **/app/** (evita loops e telas brancas)
- **Interceptor** não envia `Authorization` para `/auth/login`
- **Sem httpBasic** no Spring Security (evita popup nativo do navegador)

---

## Checklist de Status
### ✅ Concluído
- [x] Backend **rodando** (Spring Boot 3 + JWT)
- [x] Rotas `/auth/**` liberadas e `/api/**` protegidas
- [x] Front **rodando** (Angular 17)
- [x] Proxy Angular para `/auth` e `/api`
- [x] Login funcional (form + AuthService + Interceptor)
- [x] Header com saudação (email decodificado do JWT) e **Logout**
- [x] **Guard** com `canMatch` protegendo **/app/**
- [x] Tratamento de **401** (expired) e **403** (forbidden) no interceptor
- [x] Redirecionamento pós-login com `returnUrl` (fallback `/app/dashboard`)
- [x] **Auto-logout** por expiração do JWT (timer no AuthService)

### 🟨 Em Andamento / Pendências
- [ ] **Dashboard MVP** (cards: total de candidatos, novos, tecnologias/idiomas mais comuns; atalhos)
- [ ] **Talents** – refinar lista/filtros/paginação server-side
- [ ] **Roles/Perfis**:
  - Back: `@PreAuthorize("hasRole('ADMIN')")` em endpoints sensíveis
  - Front: esconder menus e bloquear rotas por `role`
- [ ] **Persistência Oracle** (adapter `infrastructure/persistence/oracle/**`)
- [ ] **Testes**: Integração (Spring) e Unit (Angular)
- [ ] **Observabilidade**: logs estruturados, healthchecks, métricas básicas (Actuator)
- [ ] **CI/CD**: pipeline para build, lint, test, package

---

## Próximos Passos Sugeridos
1. **Dashboard MVP**  
   Endpoint no back para métricas (ex.: `/api/metrics/overview`)  
   Cards + lista “Recentes” no front.

2. **Roles (ADMIN/USER)**  
   SecurityConfig + anotações `@PreAuthorize`  
   Guard por role no front e `*ngIf` no menu.

3. **Adapter Oracle**  
   Implementar `CandidatoRepositoryOracle`  
   Parametrizar datasource via `TC_DB_*`.

4. **Qualidade**  
   Lints/formatters (Java/TS), testes de integração (MockMvc/JJWT) e unit no Angular.

---

## Troubleshooting Rápido
- **Popup de usuário/senha no navegador**  
  Remover `httpBasic()` no SecurityConfig.

- **401 no `/auth/login`**  
  `JwtAuthFilter.shouldNotFilter` precisa **ignorar** `/auth/**`.  
  Interceptor **não** deve anexar Bearer no login.

- **Tela branca após separar `/app/**`**  
  Links do header devem usar `/app/dashboard` e `/app/talents`.  
  Evitar `canMatch` em `path: ''` global; encapsular área segura em `/app`.

- **403 inesperado**  
  Verificar se backend retornou 403 intencionalmente (roles).  
  Interceptor redireciona para `/dashboard?reason=forbidden`.

---

## Licença
Projeto interno de estudo/POC — ajustar conforme política da empresa.
