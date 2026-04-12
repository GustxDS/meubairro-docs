---
tags: [spec, auth, jwt, session, security, refresh-token]
aliases: [Authentication, Login, JWT, Auth Spec]
---

# Authentication Specification

## 1. Visão Geral

O sistema utiliza autenticação baseada em e-mail e senha, com gerenciamento de sessão via JWT (JSON Web Token).

A autenticação é obrigatória para acesso às funcionalidades do aplicativo.

---

## 2. Métodos de Autenticação

O sistema deve suportar:

- Cadastro de usuário (e-mail e senha)
- Login de usuário existente

---

## 3. Cadastro de Usuário

### 3.1 Dados obrigatórios

- email (único)
- senha

---

### 3.2 Regras

- o e-mail deve ser único no sistema
- a senha deve ser armazenada de forma segura (hash)
- não armazenar senha em texto puro

---

### 3.3 Processo

1. Usuário informa e-mail e senha
2. Sistema valida os dados
3. Sistema cria o usuário
4. Sistema retorna um token JWT (login automático opcional)

---

## 4. Login

### 4.1 Dados obrigatórios

- email
- senha

---

### 4.2 Processo

1. Usuário envia credenciais
2. Sistema valida:
   - e-mail existente
   - senha correta (comparação com hash)
3. Sistema gera um JWT Principal (Access Token) e um Refresh Token
4. O Frontend armazena silenciosamente os tokens via `expo-secure-store` e prossegue

---

## 5. JWT (JSON Web Token)

### 5.1 Objetivo

- Identificar o usuário autenticado
- Permitir acesso a rotas protegidas

---

### 5.2 Estrutura do Token

O token deve conter informações mínimas:

- user_id
- email

Exemplo de payload:

```json
{
  "user_id": "uuid",
  "email": "user@email.com"
}
```

---

### 5.3 Regras

- o token deve ter tempo de expiração
- o token deve ser assinado com uma chave segura
- não armazenar dados sensíveis no token

---

### 5.4 Uso

O token deve ser enviado em todas as requisições autenticadas:

`Authorization: Bearer <token>`

---

## 6. Sessão e Refresh Tokens

### 6.1 Comportamento

- o cliente (mobile via Expo / Web) deve armazenar o Access Token (`auth_token`) e o Refresh Token (`refresh_token`) no SecureStore/localStorage.
- O Access Token (Short-Lived) garante as credenciais para acessar as rotas normais.
- O Refresh Token (Long-Lived, gerado usando criptografia segura no Backend e armazenado em Postgres) garante a revalidação da sessão.

---

### 6.2 O Interceptor de Rotas

O ambiente de API Rest na aplicação cliente contém um `Axios Response Interceptor`:
1. Uma API retorna HTTP 401 (Unauthorized) devido à expiração do Access Token.
2. O Axios intercepta isso, não quebrando a experiência de usuário.
3. O cliente chama invisivelmente a rota de `/api/auth/refresh` possuindo o Refresh Token.
4. O Backend estipula um Access Token recém-renovado.
5. O cliente executa um retry silencioso na rota do item #1.

---

### 6.3 Expiração de Tokens

- O Access Token possui ciclo de vida encurtado (ex: 7 dias) mitigando risco de roubo.
- O Refresh Token possui cronograma dilatado e persistente (30 dias) e confere segurança contínua e sem fricção na UX. Na expiração, um logout definitivo ocorre.

---

## 7. Rotas Protegidas

### Regras

- todas as rotas que acessam dados do sistema devem exigir autenticação
- o backend deve validar o token em todas as requisições protegidas

---

## 8. Estado do Usuário no Sistema

Após autenticação, o sistema deve identificar:

- se o usuário está vinculado a um bairro
- se o usuário possui solicitação pendente
- qual é o papel do usuário (role)

---

## 9. Fluxo Pós-Login

Após login, o sistema deve direcionar o usuário com base no estado:

### 9.1 Usuário sem bairro

Exibir tela de:
- criar bairro
- entrar em bairro

---

### 9.2 Usuário com solicitação pendente

- exibir status de aprovação
- bloquear acesso ao conteúdo até aprovação

---

### 9.3 Usuário ativo em um bairro

- direcionar para a tela principal (Status do Bairro)

---

## 10. Segurança

### Regras obrigatórias

- senhas devem ser armazenadas com hash (ex: bcrypt)
- nunca retornar senha na API
- validar token em todas as rotas protegidas
- não confiar em dados vindos do cliente

---

## 11. Logout

### Comportamento

- o cliente remove seus dois tokens (`auth_token`, `refresh_token`) da memória em disco.
- o Backend engatilha um "Revoke" contra o Refresh Token, quebrando e invalidando reuso via API.

---

## 12. Observações de Implementação

- recomenda-se uso de middleware para validação de JWT
- autenticação deve ser centralizada no backend
- o frontend deve apenas armazenar e enviar o token

---

## 13. Possíveis Evoluções Futuras

- login social (Google, Apple)
- recuperação de senha (reset via e-mail)
- verificação de e-mail