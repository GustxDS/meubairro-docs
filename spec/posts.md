---
tags: [spec, posts, avisos, negocios, content, expiration]
aliases: [Posts, Avisos, Negócios, Posts Spec]
---

# Posts Specification

## 1. Visão Geral

O sistema possui dois tipos principais de posts:

- avisos (comunicação estruturada)
- negócios (anúncios de produtos e serviços)

Todos os posts pertencem a um bairro e só podem ser visualizados por membros desse bairro.

---

## 2. Tipos de Post

### 2.1 Avisos

Posts voltados para comunicação entre moradores, com contexto e descrição detalhada.

#### Objetivo:
Informar situações relevantes do bairro de forma organizada.

---

### 2.2 Negócios

Posts voltados para venda de produtos ou oferta de serviços locais.

#### Objetivo:
Facilitar a comunicação comercial entre moradores sem poluir os avisos.

---

## 3. Estrutura Geral de um Post

Campos comuns a todos os posts:

- id
- tipo (aviso | negocio)
- titulo
- descricao
- autor_id
- bairro_id
- data_criacao
- data_expiracao
- ativo (boolean)

Campos opcionais:

- imagem_url
- contato (apenas para negócios)

---

## 4. Avisos

### 4.1 Categorias

Cada aviso deve pertencer a uma categoria:

- seguranca
- utilidade_publica
- eventos

---

### 4.2 Estrutura

Campos específicos:

- categoria
- (opcional) imagem

---

### 4.3 Regras de Expiração

A expiração depende da categoria:

- seguranca: 6 a 12 horas
- utilidade_publica: 4 a 24 horas
- eventos: até data do evento

A data de expiração deve ser definida no momento da criação.

---

### 4.4 Comportamento

- Avisos expiram automaticamente
- Avisos expirados não devem aparecer no feed principal
- Avisos podem ser removidos por admin ou superadmin

---

## 5. Negócios

### 5.1 Tipos

- produto (desapegos)
- servico (profissionais locais)

---

### 5.2 Estrutura

Campos específicos:

- tipo_negocio (produto | servico)
- contato (telefone ou WhatsApp)

---

### 5.3 Regras de Expiração

- Todos os posts expiram em 7 dias após criação

---

### 5.4 Interação

- Usuário pode clicar em "Tenho Interesse"
- O sistema deve abrir o WhatsApp com mensagem pré-preenchida

Exemplo de mensagem:
"Olá, vi seu anúncio no app do bairro e tenho interesse."

---

## 6. Criação de Posts

### Permissões

- membro: pode criar
- admin: pode criar
- superadmin: pode criar

---

### Regras

- usuário deve estar associado a um bairro
- todos os posts devem estar vinculados ao bairro do usuário
- data de expiração deve ser obrigatória

---

## 7. Feed de Posts

### Avisos

- ordenados por data de criação (mais recentes primeiro)
- podem ser filtrados por categoria

---

### Negócios

- ordenados por data de criação
- não possuem filtro por categoria (opcional no futuro)

---

## 8. Expiração

### Regras gerais

- posts não são deletados automaticamente, apenas marcados como inativos
- campo `ativo` deve ser atualizado após expiração
- backend deve filtrar apenas posts ativos

---

## 9. Moderação

### Admin

- pode remover qualquer post

### Superadmin

- pode remover qualquer post

---

## 10. Regras de Consistência

- todo post deve pertencer a um bairro
- todo post deve ter um autor válido
- posts expirados não devem aparecer no feed
- usuários não podem editar posts de outros usuários (opcional no MVP)

---

## 11. Observações de Implementação

- expiração pode ser tratada via:
  - verificação no backend (query)
  - ou job/cron (opcional)

- recomenda-se usar timestamps para controle de validade

---

## 12. Possíveis Evoluções Futuras

- sistema de destaque (posts fixados)
- edição de posts
- múltiplas imagens
- sistema de denúncia