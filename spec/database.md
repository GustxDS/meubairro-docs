---
tags: [spec, database, schema, postgresql, drizzle, backend]
aliases: [Database Schema, Database Spec, DB Schema]
---

# Database Specification — Meu Bairro

## 1. Visão Geral

Schema do banco de dados PostgreSQL (hospedado no Neon Serverless), modelado e conectado via Drizzle ORM. Todas as 10 tabelas utilizam UUID como chave primária e timestamps para controle de criação/atualização. As tabelas são exportadas ativamente através de um objeto no arquivo `schema.ts`.

---

## 2. Convenções

- Chaves primárias: UUID (gerado automaticamente)
- Timestamps: `created_at` com default `now()`, e `deleted_at` nativo do Drizzle suportando nulos!
- Soft delete: implementado com abrangência massiva. O objeto `deleted_at` captura timestamps de destruição e os filtra em consultas com a clausula `isNull(table.deleted_at)` e `set({ deleted_at: new Date() })` substituindo completamente deleções hard.
- Naming: snake_case para tabelas e colunas
- Relacionamentos: foreign keys com `ON DELETE CASCADE` limitadas a vínculos muito restritos (gerados por hard deletes ocasionais de contas).

---

## 3. Tabelas

### 3.1 users

Armazena dados de autenticação dos usuários.

| Coluna       | Tipo        | Constraints             |
|--------------|-------------|-------------------------|
| id           | uuid        | PK, default gen         |
| email        | varchar     | UNIQUE, NOT NULL        |
| name         | varchar(100)| NOT NULL                |
| password     | varchar     | NOT NULL (hash bcrypt)  |
| created_at   | timestamp   | NOT NULL, default now() |
| deleted_at   | timestamp   | NULL                    |

**Índices:** UNIQUE em `email`

---

### 3.2 neighborhoods

Representa um bairro (comunidade).

| Coluna       | Tipo        | Constraints             |
|--------------|-------------|-------------------------|
| id           | uuid        | PK, default gen         |
| name         | varchar     | NOT NULL                |
| city         | varchar     | NOT NULL                |
| state        | varchar     | NOT NULL                |
| cep          | varchar     | NULL                    |
| access_code  | varchar     | UNIQUE, NOT NULL        |
| invite_token | varchar     | UNIQUE, NOT NULL        |
| created_at   | timestamp   | NOT NULL, default now() |
| deleted_at   | timestamp   | NULL                    |

**Índices:** UNIQUE em `access_code`, UNIQUE em `invite_token`

---

### 3.3 neighborhood_members

Associação entre usuários e bairros (N:1). Define o role do usuário no bairro.

| Coluna           | Tipo        | Constraints                     |
|------------------|-------------|---------------------------------|
| id               | uuid        | PK, default gen                 |
| user_id          | uuid        | FK → users.id, UNIQUE, NOT NULL |
| neighborhood_id  | uuid        | FK → neighborhoods.id, NOT NULL |
| role             | varchar     | NOT NULL (membro/admin/superadmin) |
| joined_at        | timestamp   | NOT NULL, default now()         |
| deleted_at       | timestamp   | NULL                            |

**Índices:** UNIQUE em `user_id` (um usuário por bairro)

**Regras:**
- Exatamente um registro com role `superadmin` por `neighborhood_id`
- `ON DELETE CASCADE` em relação a `neighborhoods`

---

### 3.4 join_requests

Solicitações de entrada em bairro via código de acesso.

| Coluna           | Tipo        | Constraints                     |
|------------------|-------------|---------------------------------|
| id               | uuid        | PK, default gen                 |
| user_id          | uuid        | FK → users.id, NOT NULL         |
| neighborhood_id  | uuid        | FK → neighborhoods.id, NOT NULL |
| status           | varchar     | NOT NULL (pendente/aprovado/rejeitado) |
| created_at       | timestamp   | NOT NULL, default now()         |
| deleted_at       | timestamp   | NULL                            |

**Índices:** Índice composto em `(user_id, neighborhood_id, status)`

---

### 3.5 posts

Armazena avisos e negócios.

| Coluna         | Tipo        | Constraints                     |
|----------------|-------------|---------------------------------|
| id             | uuid        | PK, default gen                 |
| type           | varchar     | NOT NULL (aviso/negocio)        |
| title          | varchar     | NOT NULL                        |
| description    | text        | NOT NULL                        |
| category       | varchar     | NULL (seguranca/utilidade_publica/eventos) |
| business_type  | varchar     | NULL (produto/servico)          |
| contact        | varchar     | NULL (telefone/WhatsApp)        |
| image_url      | varchar     | NULL                            |
| author_id      | uuid        | FK → users.id, NOT NULL         |
| neighborhood_id| uuid        | FK → neighborhoods.id, NOT NULL |
| active         | boolean     | NOT NULL, default true          |
| created_at     | timestamp   | NOT NULL, default now()         |
| expires_at     | timestamp   | NOT NULL                        |
| deleted_at     | timestamp   | NULL                            |

**Índices:**
- Índice em `(neighborhood_id, type, active)`
- Índice em `expires_at` (para queries de expiração)

**Regras:**
- Se `type = aviso`: `category` é obrigatório
- Se `type = negocio`: `business_type` e `contact` são obrigatórios
- `ON DELETE CASCADE` em relação a `neighborhoods`

---

### 3.6 statuses

Status em tempo real do bairro.

| Coluna           | Tipo        | Constraints                     |
|------------------|-------------|---------------------------------|
| id               | uuid        | PK, default gen                 |
| type             | varchar     | NOT NULL (falta_agua/falta_energia/barulho_excessivo/coleta_lixo) |
| neighborhood_id  | uuid        | FK → neighborhoods.id, NOT NULL |
| total_confirmations | integer  | NOT NULL, default 0             |
| active           | boolean     | NOT NULL, default true          |
| started_at       | timestamp   | NOT NULL, default now()         |
| expires_at       | timestamp   | NOT NULL                        |
| deleted_at       | timestamp   | NULL                            |

**Índices:** Índice UNIQUE em `(neighborhood_id, type, active)` WHERE `active = true AND deleted_at IS NULL`

**Regra:** Apenas um status ativo por tipo e por bairro.

---

### 3.7 status_confirmations

Confirmações de usuários em status ativos.

| Coluna     | Tipo        | Constraints                     |
|------------|-------------|---------------------------------|
| id         | uuid        | PK, default gen                 |
| status_id  | uuid        | FK → statuses.id, NOT NULL      |
| user_id    | uuid        | FK → users.id, NOT NULL         |
| created_at | timestamp   | NOT NULL, default now()         |

**Índices:** UNIQUE em `(status_id, user_id)`

**Regra:** Um usuário pode confirmar apenas uma vez por status ativo.

---

### 3.8 refresh_tokens

Mantém a sessão segura e renovável sem expiração forçada brusca.

| Coluna     | Tipo        | Constraints                     |
|------------|-------------|---------------------------------|
| id         | uuid        | PK, default gen                 |
| user_id    | uuid        | FK → users.id, NOT NULL CASCADE |
| token      | varchar     | UNIQUE, NOT NULL                |
| expires_at | timestamp   | NOT NULL                        |
| revoked_at | timestamp   | NULL                            |
| created_at | timestamp   | NOT NULL, default now()         |

---

### 3.9 user_devices

Tokens de push notification (Expo) por dispositivo.

| Coluna           | Tipo        | Constraints                     |
|------------------|-------------|---------------------------------|
| id               | uuid        | PK, default gen                 |
| user_id          | uuid        | FK → users.id, NOT NULL CASCADE |
| expo_push_token  | varchar(200)| UNIQUE, NOT NULL                |
| platform         | varchar(10) | NOT NULL (ios/android)          |
| last_seen_at     | timestamp   | NOT NULL, default now()         |
| created_at       | timestamp   | NOT NULL, default now()         |

**Regras:**
- `expo_push_token` é UNIQUE globalmente
- Hard delete no logout ou erro `DeviceNotRegistered` do Expo
- `last_seen_at` atualizado em cada upsert do token

---

### 3.10 user_notifications

Histórico de notificações in-app por usuário.

| Coluna     | Tipo        | Constraints                     |
|------------|-------------|---------------------------------|
| id         | uuid        | PK, default gen                 |
| user_id    | uuid        | FK → users.id, NOT NULL CASCADE |
| action     | varchar(30) | NOT NULL (POST_CREATED/STATUS_EXPIRED) |
| title      | varchar(255)| NULL                            |
| body       | text        | NULL                            |
| data       | jsonb       | NULL ({ postType, statusType, neighborhoodId }) |
| read_at    | timestamp   | NULL                            |
| created_at | timestamp   | NOT NULL, default now()         |

**Índices:** Índice em `(user_id, created_at)`

**Regras:**
- Criada automaticamente pelo `pushService.sendToNeighborhood()` ao enviar push
- `read_at` é preenchido em batch via `PATCH /api/notifications/read-all`

---

## 4. Diagrama de Relacionamentos

```
users
  │
  ├──< neighborhood_members >──┐
  │                             │
  ├──< join_requests >──────── neighborhoods
  │                             │
  ├──< posts >─────────────────┘
  │                             │
  ├──< status_confirmations     │
  │         │                   │
  │         └──> statuses >─────┘
  │
  ├──< user_devices
  │
  ├──< user_notifications
  │
```

**Legenda:** `──<` = um para muitos, `>──` = muitos para um

---

## 5. Tipos Drizzle (Referência)

```ts
// Enums
type Role = "membro" | "admin" | "superadmin";

type PostType = "aviso" | "negocio";

type NoticeCategory = "seguranca" | "utilidade_publica" | "eventos";

type BusinessType = "produto" | "servico";

type StatusType =
  | "falta_agua"
  | "falta_energia"
  | "barulho_excessivo"
  | "coleta_lixo";

type JoinRequestStatus = "pendente" | "aprovado" | "rejeitado";
```

---

## 6. Regras de Integridade

### Cascade

- Deletar `neighborhood` → remove todos os `neighborhood_members`, `posts`, `statuses`, `join_requests` associados
- Deletar `user` → remove `neighborhood_members`, `posts`, `status_confirmations`, `refresh_tokens`, `user_devices`, `user_notifications` associados
- Deletar `status` → remove todos os `status_confirmations` associados

---

### Constraints de Negócio

- `neighborhood_members.user_id` é UNIQUE (um bairro por usuário)
- `statuses` com `active = true` é UNIQUE por `(neighborhood_id, type)`
- `status_confirmations` é UNIQUE por `(status_id, user_id)`
- Deve existir exatamente um `neighborhood_members` com `role = superadmin` por `neighborhood_id`

---

## 7. Migrações e Seeding

- Migrations são gerenciadas pelo Drizzle Kit (`npx drizzle-kit push`) ou via painel do Neon.
- Armazenadas e referenciadas dinamicamente nas tabelas no `src/db/schema.ts` onde todas as constraints de arrays foram refatoradas em objetos literais (`PgTableExtraConfig`) para satisfazer o TypeScript.
- **Seeding:** O ambiente vem com um robusto utilitário de seeding (`yarn db:seed` que executa `seed.ts`). Este utilitário zera deterministicamente os dados do Workspace e popular tabelas vitais para testes locais: superadmins predefinidos, avisos randomizados da comunidade, e injeção de status pré-confirmados.

---

## 8. Possíveis Evoluções Futuras

- Campo `avatar_url` em `users`
- Tabela `reports` para sistema de denúncias
- Suporte a múltiplas imagens por post (tabela `post_images`)
- Histórico de status (manter registros expirados para analytics)
