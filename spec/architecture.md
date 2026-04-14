---
tags: [spec, architecture, tech-stack, react-native, expo, nodejs, postgresql, drizzle]
aliases: [Architecture, Tech Stack, Architecture Spec]
---

# Architecture Specification — Meu Bairro

## 1. Visão Geral

Este documento descreve a arquitetura completa do projeto Meu Bairro, incluindo as tecnologias utilizadas, estrutura de repositórios, padrões de comunicação, organização de código e fluxo de dados.

---

## 2. Stack Tecnológica

### 2.1 Frontend

| Tecnologia                   | Finalidade Aplicada                                                               |
|------------------------------|-----------------------------------------------------------------------------------|
| **React Native** (`0.81.5`)  | O core engine fornecendo o runtime para renderizar nativamente interfaces mobile. |
| **Expo** (`54.0.x`)          | Framework base. Gerencia plugins nativos, build OTA e ferramentas de desenvolvimento.|
| **Expo Router**              | Cria o mapeamento das rotas baseadas na estrutura das pastas filhas dentro de `app/`. |
| **NativeWind & TailwindCSS** | Processador bridge das propriedades de estilização em classes nativas aplicáveis. |
| **Zustand + Immer**          | Estado local persistido (permissions store) com mutações imutáveis via MMKV.      |
| **MMKV** (`react-native-mmkv`)| Storage síncrono AES-256 para preferências e flags (nunca tokens).               |
| **Lucide-React-Native**      | Motor de vetores iconográficos da UI moderna injetada nos cabeçalhos e status.    |
| **React Hook Form**          | Gerencia a lógica profunda de mutação do forms e feedback dos erros UX em tempo real.|
| **Zod / @hookform**          | Declara o Schema restrito que valida todo input de formulário dinamicamente antes do fetch.|
| **@tanstack/react-query** v5 | Gerencia estado remoto: queries, mutations, cache, optimistic updates e invalidação.|
| **Axios**                    | O Wrapper injetor que impulsiona os requests disparando tokens salvos no SecureStore. |
| **@sentry/react-native**    | Captura exceções em produção, rastreamento de usuário e performance tracing (20%). |
| **expo-notifications**       | Push notifications (Expo push token), handlers foreground/tap, canais Android.     |
| **React Native Reanimated**  | Gerencia o frame-loop permitindo micro-animações visuais fluidas à 60FPS.         |
| **Expo Secure Store**        | Motor criptografado para preservar seus tokens de sessão persistindo reinícios.   |
| **Clsx & Tailwind-Merge**    | Utilitários agnósticos controlando condicionais visíveis fundindo blocos no Tailwind.|
| **TypeScript**               | O contrato forte cobrindo toda a comunicação evitando reflets mal tipados na View.|

---

### 2.2 Backend

| Tecnologia                   | Finalidade Aplicada                                                               |
|------------------------------|-----------------------------------------------------------------------------------|
| **Node.js + Express**        | Listener REST escalável processando todas as requisições roteadas até os services. |
| **TSX**                      | Engine para ambiente de desenvolvimento permitindo compilações TypeScript e hot-reload.|
| **Drizzle ORM**              | Core ORM Typesafe que transforma instâncias JS puras em blocos relacionais SQL.   |
| **Drizzle Kit**              | Interface CLI para criação contínua de migrations e sincronização direta no banco.|
| **Postgres**                 | Driver subjacente e nativo garantindo o link forte entre o ORM e o banco no Neon. |
| **Zod**                      | Camada bruta DTO validando toda injeção suspeita no Body das requisições Express. |
| **JsonWebToken**             | Estratégia criptográfica gerando instâncias stateless de autorização nas rotas.   |
| **Bcrypt**                   | Salting seguro, efetuando hash de senhas de forma assimétrica antes das inserções.|
| **Multer**                   | Interceptação eficiente parseando dados Binários provindos do fluxo multipart HTTP. |
| **Cors**                     | Proxy wrapper blindando origens web se o app necessitar invocar instâncias remotas.|
| **Helmet**                   | Interceptador Middleware injetando headers de segurança globais.                  |
| **TypeScript**               | Amarrações de dados estritas casando parâmetros provindos do Drizzle nas Entidades.|

---

### 2.3 Banco de Dados

| Tecnologia         | Finalidade                                |
|--------------------|-------------------------------------------|
| PostgreSQL         | Banco relacional principal                |

Suporte a tipos avançados: UUID, JSONB, timestamps com timezone.

---

### 2.4 Integrações Externas

| Serviço            | Finalidade                                |
|--------------------|-------------------------------------------|
| BrasilAPI          | Consulta de CEP e dados de endereço       |
| Expo Push Service  | Envio de push notifications via expo-server-sdk |
| Sentry             | Error tracking e performance monitoring em produção |

BrasilAPI não requer autenticação. Expo Push usa o projectId do EAS. Sentry usa DSN via variável de ambiente.

---

### 2.5 Infraestrutura de Desenvolvimento

| Ferramenta         | Finalidade                                |
|--------------------|-------------------------------------------|
| Android Studio     | Android SDK e emuladores                  |
| Expo Go            | Testes em dispositivo físico com hot reload |

---

## 3. Estrutura de Repositórios

O projeto utiliza **dois repositórios separados**:

```
meubairro/    → Frontend (React Native + Expo)
meubairro-backend/    → Backend (Node.js + Drizzle + PostgreSQL)
```

### Justificativa

- Deploys independentes entre frontend e backend
- Cada repositório possui seu próprio `package.json`, scripts e configurações
- Expo funciona melhor como repositório standalone
- Git history focado e limpo por camada
- Tipos são mantidos em sincronia manualmente (suficiente para MVP)

---

## 4. Estrutura de Pastas

### 4.1 Frontend (`meubairro/`)

```
meubairro/
├── app/                              # Expo Router (telas e navegação)
│   ├── _layout.tsx                   # Root Layout (auth guard, Sentry, ErrorBoundary, QueryClient)
│   ├── index.tsx                     # Splash / Redirect
│   ├── (auth)/                       # Telas públicas (login, register)
│   ├── (onboarding)/                 # Fluxo de associação ao bairro
│   └── (app)/                        # Área autenticada
│       ├── _layout.tsx               # Stack (notificações, offline banner)
│       ├── notifications/index.tsx   # Histórico de notificações
│       └── (tabs)/                   # Tab navigator (4 tabs)
│           ├── _layout.tsx           # Tab navigator config
│           ├── status/index.tsx
│           ├── notices/              # index.tsx + create.tsx
│           ├── business/             # index.tsx + create.tsx
│           └── profile/              # index.tsx + members.tsx
│
├── components/                       # Componentes reutilizáveis
│   ├── ui/                           # Componentes RNR (copiados)
│   │   └── index.ts                  # barrel: Button, Input, Card, Badge, Text, Separator
│   ├── custom/                       # Componentes do projeto
│   │   └── index.ts                  # barrel: Header, PostCard, StatusCard, NotificationBell…
│   └── index.ts                      # root barrel (re-exports ui + custom)
│
├── lib/                              # Utilitários e configurações
│   ├── api.ts                        # Axios instance + token helpers + image upload
│   ├── constants.ts                  # STATUS_TYPES, NOTICE_CATEGORIES, ROLE_LEVEL, COLORS
│   ├── utils.ts                      # cn() (clsx+twMerge), canDeletePost()
│   ├── logger.ts                     # Structured logger (dev console / prod Sentry)
│   ├── toast.ts                      # Native toast via burnt
│   ├── storage.ts                    # MMKV instance + clientStorage adapter
│   ├── animations.ts                 # Reanimated spring/timing presets
│   └── index.ts                      # barrel: re-exports all lib modules
│
├── hooks/                            # Custom hooks
│   ├── use-neighborhood.ts           # Neighborhood data query
│   ├── use-create-notice.ts          # Notice form logic + mutation
│   ├── use-create-business.ts        # Business form logic + mutation
│   ├── use-notifications.ts          # useNotifications, useUnreadCount, useMarkAllRead
│   ├── use-net-info.ts               # Network connectivity state
│   ├── use-permissions-store.ts      # Zustand + Immer + MMKV persistence
│   ├── useImagePicker.ts             # Image selection + upload
│   ├── usePushNotifications.ts       # Push token registration
│   └── index.ts                      # barrel: re-exports all hooks
│
├── contexts/                         # React Contexts
│   ├── auth-context.tsx              # Auth state, Axios interceptor, push token lifecycle, Sentry
│   └── index.ts                      # barrel
│
├── types/                            # Tipos TypeScript compartilhados
│   └── index.ts                      # User, Post, Status, AppNotification, ApiError…
│
├── assets/                           # Imagens, fontes, ícones
│
├── tailwind.config.js                # Configuração NativeWind
├── app.config.js                     # Configuração Expo (Sentry, notifications, EAS)
├── tsconfig.json
└── package.json
```

---

### 4.2 Backend (`meubairro-backend/`)

```
meubairro-backend/
├── src/
│   ├── routes/                  # Definição de rotas por recurso
│   │   ├── auth.ts
│   │   ├── neighborhoods.ts
│   │   ├── posts.ts
│   │   ├── status.ts
│   │   ├── members.ts
│   │   ├── devices.ts           # Push token management
│   │   ├── notifications.ts     # In-app notification history
│   │   └── uploads.ts           # Upload de imagens
│   │
│   ├── middleware/               # Middlewares (auth, validação)
│   │   ├── auth.ts              # Validação de JWT
│   │   ├── validate.ts          # Validação de request body
│   │   └── upload.ts            # Multer / processamento de arquivos
│   │
│   ├── db/                      # Camada de banco de dados
│   │   ├── schema.ts            # Schema Drizzle (tabelas)
│   │   ├── index.ts             # Conexão com PostgreSQL
│   │   └── migrations/          # Migrações geradas pelo Drizzle Kit
│   │
│   ├── services/                # Lógica de negócio
│   │
│   ├── types/                   # Tipos TypeScript
│   │
│   ├── utils/                   # Utilitários (hash, tokens, etc.)
│   │
│   └── index.ts                 # Entry point do servidor
│
├── uploads/                     # Arquivos enviados (imagens de posts)
│   └── posts/                   # Imagens organizadas por recurso
│
├── drizzle.config.ts            # Configuração do Drizzle Kit
├── tsconfig.json
└── package.json
```

---

## 5. Diagrama de Arquitetura

```
┌──────────────────────────────────────────┐
│              Mobile App                  │
│  React Native + Expo + NativeWind + RNR  │
│              (TypeScript)                │
└──────────┬───────────────┬───────────────┘
           │               │ Push Notifications
           │ HTTP (REST)   │ (Expo Push Service)
           │ Bearer <JWT>  │
┌──────────▼───────────────▼───────────────┐
│              Backend API                 │
│  Node.js + TypeScript                    │
│  ┌─────────────────────────────────────┐ │
│  │ Middleware (Auth) → Routes → Services│ │
│  │         pushService (expo-server-sdk)│ │
│  │         cron (node-cron)             │ │
│  └─────────────────────────────────────┘ │
└──────────────────┬───────────────────────┘
                   │ SQL (Drizzle ORM)
┌──────────────────▼───────────────────────┐
│              PostgreSQL                  │
│  users, neighborhoods, posts, status,    │
│  confirmations, join_requests,           │
│  user_devices, user_notifications,       │
│  refresh_tokens                          │
└──────────────────────────────────────────┘

  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │  BrasilAPI   │  │    Sentry    │  │  Expo Push   │
  │ (CEP lookup) │  │ (error track)│  │  Service     │
  └──────────────┘  └──────────────┘  └──────────────┘
```

---

## 6. Fluxo de Dados

### 6.1 Ciclo de uma Requisição

```
1. Cliente envia requisição HTTP
   → Headers: Authorization: Bearer <JWT>
   → Body: JSON

2. Backend recebe a requisição
   → Middleware de auth valida o token
   → Middleware de validação verifica o body

3. Route handler processa a requisição
   → Chama service com lógica de negócio
   → Service interage com o banco via Drizzle ORM

4. Backend retorna resposta
   → Status code apropriado
   → Body: JSON padronizado
```

---

### 6.2 Fluxo de Autenticação

```
Register/Login → API gera JWT → Cliente armazena (SecureStore)
                                       │
                                       ▼
                              Requisições subsequentes
                              enviam JWT no header
                                       │
                                       ▼
                              Backend valida JWT
                              em cada requisição
```

---

## 7. Padrão de Comunicação da API

### 7.1 Base URL

```
Desenvolvimento: http://localhost:3000/api
Produção:        https://<dominio>/api
```

---

### 7.2 Convenções de Endpoints

| Método | Padrão                     | Ação                     |
|--------|----------------------------|--------------------------|
| GET    | /api/recursos              | Listar recursos          |
| GET    | /api/recursos/:id          | Obter recurso por ID     |
| POST   | /api/recursos              | Criar recurso            |
| PUT    | /api/recursos/:id          | Atualizar recurso        |
| DELETE | /api/recursos/:id          | Remover recurso          |

---

### 7.3 Formato de Resposta (Sucesso)

```json
{
  "data": { ... },
  "message": "Recurso criado com sucesso"
}
```

---

### 7.4 Formato de Resposta (Erro)

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "E-mail já cadastrado"
  }
}
```

---

### 7.5 Códigos HTTP

| Código | Uso                              |
|--------|----------------------------------|
| 200    | Sucesso                          |
| 201    | Recurso criado                   |
| 400    | Erro de validação                |
| 401    | Não autenticado (token inválido) |
| 403    | Sem permissão (RBAC)             |
| 404    | Recurso não encontrado           |
| 500    | Erro interno do servidor         |

---

## 8. Tratamento de Erros

### 8.1 Backend

- Erros de validação retornam 400 com detalhes dos campos
- Erros de autenticação retornam 401
- Erros de permissão retornam 403
- Erros inesperados retornam 500 com mensagem genérica
- Logs detalhados no servidor, mensagens seguras para o cliente

---

### 8.2 Frontend

- Interceptor centralizado no cliente HTTP para tratar erros
- Token expirado (401) → silent refresh via refresh token → retry automático
- Refresh expirado → clears tokens → redireciona para login
- `react-error-boundary` no root layout captura erros de render → Sentry
- `lib/logger.ts` — dev: console, prod: `Sentry.captureException()`
- `lib/toast.ts` — feedback visual via `burnt` (success, error, info)
- `hooks/use-net-info.ts` — banner offline quando `!isConnected`
- Erros de validação → exibe feedback inline nos formulários

---

## 9. Ambientes

### 9.1 Desenvolvimento

```
Frontend: Expo Dev Server (npx expo start)
Backend:  Node.js local (npm run dev)
Banco:    PostgreSQL local
```

---

### 9.2 Variáveis de Ambiente

**Backend (`meubairro-backend/.env`)**

```env
DATABASE_URL=postgresql://user:pass@localhost:5432/meubairro
JWT_SECRET=chave-segura
PORT=3000
```

**Frontend (`meubairro/.env`)**

```env
EXPO_PUBLIC_API_URL=http://localhost:3000/api
SENTRY_DSN=https://<key>@sentry.io/<project>
```

---

## 10. Upload de Arquivos

### 10.1 Estratégia

Imagens são armazenadas no disco do servidor e servidas como arquivos estáticos. Sem dependência de serviços externos (S3, Cloudinary).

---

### 10.2 Fluxo

```
1. Frontend envia imagem via multipart/form-data
2. Backend recebe e valida (tipo, tamanho)
3. Backend salva em /uploads/posts/<uuid>.<ext>
4. Backend retorna a URL pública do arquivo
5. URL é armazenada no campo image_url do post
```

---

### 10.3 Armazenamento

```
uploads/
└── posts/
    ├── a1b2c3d4-foto.jpg
    └── e5f6g7h8-produto.png
```

- Arquivos renomeados com UUID para evitar colisões
- Servidos como estáticos: `GET /uploads/posts/<filename>`
- Diretório `uploads/` deve estar no `.gitignore`

---

### 10.4 Validações

| Regra                | Valor                          |
|----------------------|--------------------------------|
| Tipos permitidos     | JPEG, PNG, WebP                |
| Tamanho máximo       | 5 MB                           |
| Quantidade por post  | 1 imagem (MVP)                 |

---

### 10.5 Migração Futura

Quando necessário, migrar para armazenamento externo (S3, Cloudinary) requer apenas:
- Alterar o destino de salvamento no backend
- Alterar o prefixo da URL retornada
- Nenhuma mudança no frontend ou no schema do banco

---

## 11. Observações de Implementação

- Frontend e backend compartilham TypeScript como linguagem
- O Drizzle ORM gera tipos a partir do schema, evitando inconsistências
- O Expo simplifica o processo de build e deploy para Android e iOS
- NativeWind e RNR juntos permitem prototipagem rápida com componentes estilizados
- A API REST é stateless — toda autenticação é feita via JWT
- Validações críticas sempre no backend, frontend apenas reflete
- Imagens são salvas em disco e servidas como estáticos (sem S3)
- O Node captura eventos SIGINT/SIGTERM provendo Graceful Shutdown seguro nas conexões de banco e I/O

---

## 12. Possíveis Evoluções Futuras

- Cache com Redis
- CI/CD para build e deploy automatizado
- Containerização com Docker
- Monorepo com compartilhamento de tipos (quando a complexidade justificar)
- Rate limiting e proteção contra abuso
- Migração de uploads para S3/Cloudinary

> **Nota:** Push notifications já substituíram a necessidade de WebSockets para o MVP. Updates em tempo real funcionam via invalidação de cache React Query disparada por notificações push.
