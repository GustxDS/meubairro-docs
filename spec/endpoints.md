---
tags: [spec, api, endpoints, rest, backend]
aliases: [API Endpoints, Endpoints, REST API]
---

# Endpoints Specification — Meu Bairro API

## 1. Visão Geral

Referência centralizada de todos os endpoints REST da API. Cada endpoint segue o padrão de resposta definido em `architecture.md`.

**Base URL:** `/api`

---

## 2. Legenda

| Símbolo | Significado                  |
|---------|------------------------------|
| [Public]      | Rota pública (sem token)     |
| [Auth]      | Rota protegida (JWT)         |
| Member      | Qualquer membro autenticado  |
| Admin      | Admin ou Superadmin          |
| Superadmin      | Apenas Superadmin            |

---

## 3. Autenticação

### [Public] POST `/api/auth/register`

Cadastro de novo usuário.

| Campo    | Tipo   | Obrigatório |
|----------|--------|-------------|
| email    | string | ✓           |
| password | string | ✓           |

**Resposta (201):**

```json
{
  "data": {
    "user": { "id": "uuid", "email": "user@email.com" },
    "token": "jwt-token"
  }
}
```

**Erros:** 400 (e-mail inválido, já cadastrado)

---

### [Public] POST `/api/auth/login`

Login de usuário existente.

| Campo    | Tipo   | Obrigatório |
|----------|--------|-------------|
| email    | string | ✓           |
| password | string | ✓           |

**Resposta (200):**

```json
{
  "data": {
    "user": { "id": "uuid", "email": "user@email.com" },
    "token": "jwt-token"
  }
}
```

**Erros:** 400 (credenciais inválidas)

---

### [Auth] GET `/api/auth/me`

Retorna dados do usuário autenticado + estado (bairro, role, pendências).

**Resposta (200):**

```json
{
  "data": {
    "id": "uuid",
    "email": "user@email.com",
    "neighborhood_id": "uuid | null",
    "role": "membro | admin | superadmin | null",
    "pending_request": true | false
  }
}
```

---

## 4. Bairros (Neighborhoods)

### [Auth: Member] POST `/api/neighborhoods`

Criar um bairro. O usuário se torna superadmin.

| Campo  | Tipo   | Obrigatório |
|--------|--------|-------------|
| name   | string | ✓           |
| cep    | string | ✓           |
| city   | string | ✓           |
| state  | string | ✓           |

**Regra:** Usuário não pode já estar vinculado a outro bairro.

**Resposta (201):**

```json
{
  "data": {
    "id": "uuid",
    "name": "Vila Esperança",
    "access_code": "ABC123",
    "invite_link": "https://..."
  }
}
```

---

### [Auth: Member] GET `/api/neighborhoods/current`

Retorna dados do bairro do usuário autenticado.

**Resposta (200):**

```json
{
  "data": {
    "id": "uuid",
    "name": "Vila Esperança",
    "city": "São Paulo",
    "state": "SP",
    "access_code": "ABC123",
    "invite_link": "https://...",
    "created_at": "timestamp"
  }
}
```

---

### [Auth: Superadmin] DELETE `/api/neighborhoods/current`

Deletar o bairro. Remove todos os vínculos e conteúdos.

**Resposta (200):** `{ "message": "Bairro deletado" }`

---

## 5. Entrada no Bairro (Join Requests)

### [Auth: Member] POST `/api/neighborhoods/join/code`

Solicitar entrada via código de acesso.

| Campo       | Tipo   | Obrigatório |
|-------------|--------|-------------|
| access_code | string | ✓           |

**Resposta (201):**

```json
{
  "data": {
    "request_id": "uuid",
    "status": "pendente"
  }
}
```

---

### [Auth] DELETE `/api/neighborhoods/join`

Cancelar uma solicitação pendente de entrada do próprio usuário logado.

**Resposta (200):** `{ "message": "Solicitação cancelada com sucesso" }`

---

### [Auth: Member] POST `/api/neighborhoods/join/link/:token`

Entrar via link de convite. Entrada imediata, sem aprovação.

**Resposta (200):**

```json
{
  "data": {
    "neighborhood_id": "uuid",
    "role": "membro"
  }
}
```

---

### [Auth: Admin] GET `/api/join-requests`

Listar solicitações pendentes do bairro.

**Resposta (200):**

```json
{
  "data": [
    {
      "id": "uuid",
      "user_email": "user@email.com",
      "status": "pendente",
      "created_at": "timestamp"
    }
  ]
}
```

---

### [Auth: Admin] PUT `/api/join-requests/:id/approve`

Aprovar solicitação de entrada.

**Resposta (200):** `{ "message": "Solicitação aprovada" }`

---

### [Auth: Admin] PUT `/api/join-requests/:id/reject`

Rejeitar solicitação de entrada.

**Resposta (200):** `{ "message": "Solicitação rejeitada" }`

---

## 6. Membros

### [Auth: Admin] GET `/api/members`

Listar membros do bairro.

**Resposta (200):**

```json
{
  "data": [
    {
      "id": "uuid",
      "email": "user@email.com",
      "role": "membro",
      "joined_at": "timestamp"
    }
  ]
}
```

---

### [Auth: Admin] PUT `/api/members/:id/role`

Alterar role de um membro (respeitando hierarquia RBAC).

| Campo | Tipo   | Obrigatório |
|-------|--------|-------------|
| role  | string | ✓           |

**Valores:** `membro`, `admin`, `superadmin`

**Regras:**
- Admin pode promover membro → admin
- Superadmin pode promover/rebaixar qualquer role
- Transferir superadmin requer confirmação

**Erros:** 403 (sem permissão na hierarquia)

---

### [Auth: Admin] DELETE `/api/members/:id`

Remover membro do bairro.

**Regras:**
- Admin pode remover membros
- Superadmin pode remover membros e admins
- Ninguém pode remover o superadmin

**Resposta (200):** `{ "message": "Membro removido" }`

---

### [Auth: Member] POST `/api/members/leave`

Usuário sai do bairro voluntariamente.

**Regra:** Superadmin não pode sair sem transferir a função.

**Resposta (200):** `{ "message": "Você saiu do bairro" }`

---

## 7. Posts — Avisos

### [Auth: Member] GET `/api/posts/notices`

Listar avisos ativos do bairro.

**Query params:**

| Param    | Tipo   | Opcional | Descrição                         |
|----------|--------|----------|-----------------------------------|
| category | string | ✓        | seguranca, utilidade_publica, eventos |

**Resposta (200):**

```json
{
  "data": [
    {
      "id": "uuid",
      "title": "Rua interditada",
      "description": "...",
      "category": "utilidade_publica",
      "image_url": "https://... | null",
      "author": { "id": "uuid", "email": "..." },
      "created_at": "timestamp",
      "expires_at": "timestamp"
    }
  ]
}
```

---

### [Auth: Member] POST `/api/posts/notices`

Criar um aviso.

| Campo       | Tipo   | Obrigatório |
|-------------|--------|-------------|
| title       | string | ✓           |
| description | string | ✓           |
| category    | string | ✓           |
| expires_at  | string | ✓           |
| image_url   | string | ✗           |

**Resposta (201):** `{ "data": { ... } }`

---

### [Auth: Admin] DELETE `/api/posts/notices/:id`

Remover um aviso (moderação).

**Resposta (200):** `{ "message": "Aviso removido" }`

---

## 8. Posts — Negócios

### [Auth: Member] GET `/api/posts/business`

Listar negócios ativos do bairro.

**Resposta (200):**

```json
{
  "data": [
    {
      "id": "uuid",
      "title": "Geladeira semi-nova",
      "description": "...",
      "business_type": "produto",
      "contact": "11999999999",
      "image_url": "https://... | null",
      "author": { "id": "uuid", "email": "..." },
      "created_at": "timestamp",
      "expires_at": "timestamp"
    }
  ]
}
```

---

### [Auth: Member] POST `/api/posts/business`

Criar um negócio.

| Campo         | Tipo   | Obrigatório |
|---------------|--------|-------------|
| title         | string | ✓           |
| description   | string | ✓           |
| business_type | string | ✓           |
| contact       | string | ✓           |
| image_url     | string | ✗           |

**Expiração:** 7 dias automáticos (definido pelo backend).

**Resposta (201):** `{ "data": { ... } }`

---

### [Auth: Admin] DELETE `/api/posts/business/:id`

Remover um negócio (moderação).

**Resposta (200):** `{ "message": "Negócio removido" }`

---

## 9. Upload de Imagens

### [Auth: Member] POST `/api/uploads`

Upload de imagem para uso em posts. Enviado como `multipart/form-data`.

| Campo | Tipo   | Obrigatório | Descrição               |
|-------|--------|-------------|-------------------------|
| file  | File   | ✓           | Imagem (JPEG, PNG, WebP)|

**Validações (Multer):**
- Tamanho máximo: 5 MB
- Tipos permitidos: `image/jpeg`, `image/png`, `image/webp`

**Resposta (201):**

```json
{
  "data": {
    "url": "/uploads/posts/a1b2c3d4-foto.jpg"
  }
}
```

**Erros:**
- 400 (tipo inválido, tamanho excedido)
- 401 (não autenticado)

**Uso:** O frontend faz upload primeiro, recebe a URL, e envia a URL no campo `image_url` ao criar o post.

---

## 10. Status do Bairro (Realtime)

### [Auth: Member] GET `/api/status`

Listar todos os status do bairro (ativos e expirados).

**Resposta (200):**

```json
{
  "data": [
    {
      "id": "uuid",
      "type": "falta_agua",
      "total_confirmations": 5,
      "user_confirmed": true,
      "started_at": "timestamp",
      "expires_at": "timestamp",
      "active": true
    }
  ]
}
```

---

### [Auth: Member] POST `/api/status/:type/confirm`

Confirmar um status. Cria o status automaticamente se não existir.

**Tipos válidos:** `falta_agua`, `falta_energia`, `barulho_excessivo`, `coleta_lixo`

**Regras:**
- Apenas uma confirmação por usuário por status ativo.
- Se a requisição atingir um status inativo, o endpoint irá "acordar" (wake up) o status reativando-o para que os usuários possam usar o botão "Reportar" na UI Otimista no frontend.
- Se não existe status no banco daquele tipo, o sistema cria um novo.

**Resposta (200):**

```json
{
  "data": {
    "status_id": "uuid",
    "total_confirmations": 6
  }
}
```

---

### [Auth: Member] DELETE `/api/status/:type/confirm`

Remover confirmação do usuário.

**Resposta (200):**

```json
{
  "data": {
    "status_id": "uuid",
    "total_confirmations": 5
  }
}
```

---

## 11. Resumo de Endpoints

| Método | Endpoint                          | Acesso | Descrição                  |
|--------|-----------------------------------|--------|----------------------------|
| POST   | /api/auth/register                | [Public]     | Cadastro                   |
| POST   | /api/auth/login                   | [Public]     | Login                      |
| GET    | /api/auth/me                      | [Auth]     | Dados do usuário           |
| POST   | /api/neighborhoods                | [Auth: Member]   | Criar bairro               |
| GET    | /api/neighborhoods/current        | [Auth: Member]   | Dados do bairro            |
| DELETE | /api/neighborhoods/current        | [Auth: Superadmin]   | Deletar bairro             |
| POST   | /api/neighborhoods/join/code      | [Auth: Member]   | Solicitar entrada (código) |
| POST   | /api/neighborhoods/join/link/:t   | [Auth: Member]   | Entrar via link            |
| GET    | /api/join-requests                | [Auth: Admin]   | Listar solicitações        |
| PUT    | /api/join-requests/:id/approve    | [Auth: Admin]   | Aprovar solicitação        |
| PUT    | /api/join-requests/:id/reject     | [Auth: Admin]   | Rejeitar solicitação       |
| GET    | /api/members                      | [Auth: Admin]   | Listar membros             |
| PUT    | /api/members/:id/role             | [Auth: Admin]   | Alterar role               |
| DELETE | /api/members/:id                  | [Auth: Admin]   | Remover membro             |
| POST   | /api/members/leave                | [Auth: Member]   | Sair do bairro             |
| GET    | /api/posts/notices                | [Auth: Member]   | Listar avisos              |
| POST   | /api/posts/notices                | [Auth: Member]   | Criar aviso                |
| DELETE | /api/posts/notices/:id            | [Auth: Admin]   | Remover aviso              |
| GET    | /api/posts/business               | [Auth: Member]   | Listar negócios            |
| POST   | /api/posts/business               | [Auth: Member]   | Criar negócio              |
| DELETE | /api/posts/business/:id           | [Auth: Admin]   | Remover negócio            |
| POST   | /api/uploads                      | [Auth: Member]   | Upload de imagem           |
| GET    | /api/status                       | [Auth: Member]   | Listar status              |
| POST   | /api/status/:type/confirm         | [Auth: Member]   | Confirmar status           |
| DELETE | /api/status/:type/confirm         | [Auth: Member]   | Remover confirmação        |
