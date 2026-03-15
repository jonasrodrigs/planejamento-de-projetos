
# Talent Core — Frontend (Angular 17 Standalone)

> **Status**: MVP funcional (Auth + Rotas protegidas + Dashboard + Talentos + Detalhe estilo currículo)

Este documento descreve **apenas o Frontend** do projeto Talent Core, pronto para ser publicado no GitHub do repositório do front.

---

## Sumário
- [Visão Geral](#visão-geral)
- [Stack e Versões](#stack-e-versões)
- [Estrutura de Pastas](#estrutura-de-pastas)
- [Setup & Execução](#setup--execução)
- [Ambientes & Proxy](#ambientes--proxy)
- [Roteamento (Standalone)](#roteamento-standalone)
- [Autenticação & Segurança](#autenticação--segurança)
- [Páginas e Funcionalidades](#páginas-e-funcionalidades)
  - [Header (Toolbar)](#header-toolbar)
  - [Login](#login)
  - [Dashboard](#dashboard)
  - [Talentos — Lista](#talentos--lista)
  - [Talento — Detalhe (formato currículo)](#talento--detalhe-formato-currículo)
- [Padrões de Código & UI](#padrões-de-código--ui)
- [Acessibilidade & Responsividade](#acessibilidade--responsividade)
- [Boas Práticas de Commit](#boas-práticas-de-commit)
- [Troubleshooting](#troubleshooting)

---

## Visão Geral
Frontend em **Angular 17** (componentes **standalone**), com autenticação via **JWT**, rotas protegidas e design minimalista. O projeto utiliza **canMatch** para proteger o escopo **/app/** e um **interceptor** para anexar o token e tratar **401/403**.

**Fluxo básico**: Login → /app/dashboard → /app/talents → /app/talents/:id (detalhe estilo currículo).

---

## Stack e Versões
- **Angular** 17 (standalone)
- **TypeScript** 5+
- **RxJS** 7+
- **Node** 18 LTS

---

## Estrutura de Pastas
```
src/app/
├─ app.routes.ts                     # Rotas públicas e /app/**
├─ app.config.ts                     # Providers (Router, HttpClient, Interceptor)
├─ core/
│  ├─ services/
│  │  └─ auth.service.ts            # Login, token, auto-logout por expiração
│  ├─ interceptors/
│  │  └─ auth.interceptor.ts        # Bearer + 401/403
│  └─ guards/
│     └─ auth.guard.ts              # canMatch para /app
├─ shared/
│  └─ layout/
│     ├─ header-toolbar/
│     │  ├─ header-toolbar.component.ts
│     │  ├─ header-toolbar.component.html
│     │  └─ header-toolbar.component.scss
│     └─ main-layout/               # (se aplicável no projeto)
├─ features/
│  ├─ auth/
│  │  └─ pages/login.page.component.*
│  ├─ dashboard/
│  │  └─ pages/dashboard-home.component.*
│  └─ talents/
│     ├─ talents.routes.ts
│     ├─ talent-list/
│     │  ├─ talent-list.component.ts
│     │  ├─ talent-list.component.html
│     │  └─ talent-list.component.scss
│     └─ pages/talent-detail/
│        ├─ talent-detail.component.ts
│        ├─ talent-detail.component.html
│        └─ talent-detail.component.scss
└─ environments/ (se aplicável)
```

---

## Setup & Execução
```bash
# instalar dependências
npm install

# rodar em dev (com proxy para a API)
ng serve --proxy-config proxy.conf.json

# build de produção
ng build --configuration production
```

> **Requisitos**: Node 18+, Angular CLI 17+.

---

## Ambientes & Proxy
**Proxy (dev)**: `proxy.conf.json`
```json
{
  "/api":  { "target": "http://localhost:8080", "secure": false, "changeOrigin": true },
  "/auth": { "target": "http://localhost:8080", "secure": false, "changeOrigin": true }
}
```
**Environments** (opcional): `src/environments/environment.ts`
```ts
export const environment = { production: false, apiUrl: '' };
```

---

## Roteamento (Standalone)
- **Público**: `/login`
- **Área autenticada**: prefixo **`/app`** com `canMatch` para bloquear rotas sem token
  - `/app/dashboard`
  - `/app/talents`
  - `/app/talents/:id`

**Retorno após login**: suporta `?returnUrl=`. 

---

## Autenticação & Segurança
- **AuthService**: login (`/auth/login`), guarda token (localStorage/sessionStorage), `isAuthenticated`, auto‑logout por `exp` do JWT.
- **Interceptor**: anexa `Authorization: Bearer <token>` (exceto no login); trata **401** (redireciona para `/login?reason=expired`) e **403** (para `/dashboard?reason=forbidden`).
- **Header**: botão **Sair** (limpa token e volta ao login). Busca no header aparece **somente na rota /app/talents/**.

---

## Páginas e Funcionalidades

### Header (Toolbar)
- Links: **Dashboard** / **Talentos**
- **Saudação** com e‑mail do usuário (lido do `sub` do JWT)
- **Busca** (campo `q`) **apenas** em **/app/talents** → navega para `/app/talents?q=<termo>`

### Login
- Reactive Forms: campos e‑mail/senha, *remember me*
- `returnUrl` para voltar à rota protegida após autenticação
- Mostra mensagem amigável conforme `?reason=expired|forbidden|logout`

### Dashboard
- **Cards**: Total de candidatos, **Top Ocupações** (universal), Top Idiomas
- **Recentes**: tabela compacta, mesma largura da Lista de Talentos
- Defensivo: se `ocupacao` não estiver no payload, mostra “Sem dados suficientes.”

### Talentos — Lista
- **Barra de busca** no header (global da rota) → parâmetro `q`
- Colunas: **Nome | E‑mail | Cidade/UF | Ocupação | Idiomas | Ações**
- **Responsivo real**: rolagem horizontal em telas estreitas (não esconde colunas)
- Ação **Detalhes** por linha

### Talento — Detalhe (formato currículo)
- Layout tipo **currículo**: Nome (destaque) + **Ocupação** (opcional)
- **Contato**: e‑mail (ellipsis + title), telefone
- **Localização**: Cidade/UF/Pais
- **Idiomas**: chips
- **Tecnologias**: chips (se existirem)
- **Botões “Voltar”** no topo e no rodapé
- **Print‑friendly** (preto e branco com bordas nítidas)
- Campos planejados (front‑only, opcionais): `fotoUrl`, `resumoProfissional`, `linkedin`, `github`, `portfolio`, `logradouro`, `numero`, `complemento`, `bairro`, `cep`, `dataNascimento`, `nacionalidade`, `estadoCivil`, `aceitaViagens`, `aceitaMudanca`, `horarios`, `pretensaoSalarial`.

> *Importante*: Atualmente o backend **ainda não** envia `ocupacao` (campo universal); o front trata como **opcional** e exibe `—` quando ausente.

---

## Padrões de Código & UI
- Componentes **standalone**; evitar módulos desnecessários
- Estilos **minimalistas** e **consistentes**; sem libs de UI pesadas
- **Table layout fixed** + truncamento (ellipsis) em colunas longas
- Evitar lógica pesada no template; usar *signals/computed* ou getters simples

---

## Acessibilidade & Responsividade
- Tabelas com `role="region"` + `aria-label` e foco (`tabindex="0"`)
- Botões e links com `aria-label`/`title` onde fizer sentido
- Layouts com **max-width: 980px** (conteúdo centralizado) e rolagem horizontal para tabelas no mobile

---

## Boas Práticas de Commit
- Commits atômicos e descritivos: `feat(talents): add detail page (cv layout)`
- PRs curtos com antes/depois quando alterar UI

---

## Troubleshooting
- **Voltou para /login ao clicar em Talentos/Dashboard?**
  - Links do header devem usar `/app/...` (não `/talents`/`/dashboard` sem prefixo)
- **Tela branca após split /app**
  - Use `canMatch` **apenas** no grupo `/app`; evite `canMatch` em `path: ''` global
- **401/403 inesperado**
  - Confirme token válido; verifique interceptor e perfis/roles no backend
- **Busca não retorna resultados**
  - Verifique se o backend trata `?q=` ou se o termo existe nos campos pesquisáveis

---

## Licença
Projeto interno/POC. Ajuste conforme a política da empresa antes de torná-lo público.
