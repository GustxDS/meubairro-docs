---
tags: [spec, rbac, roles, permissions, auth]
aliases: [Roles, RBAC, Permissions, Roles Spec]
---

# Roles and Permissions Specification

## 1. Visão Geral

O sistema utiliza um modelo de controle de acesso baseado em papéis (RBAC — Role-Based Access Control).

Cada usuário dentro de um bairro possui exatamente um dos seguintes papéis:

- membro
- admin
- superadmin

As permissões são definidas com base nesse papel.

---

## 2. Hierarquia de Permissões

A hierarquia deve seguir a seguinte ordem:

superadmin > admin > membro

### Regras gerais:

- Um usuário só pode executar ações sobre usuários com nível inferior ao seu
- Usuários não podem executar ações sobre outros com o mesmo nível
- Sempre deve existir exatamente um superadmin por bairro

---

## 3. Definição de Roles

### 3.1 Membro

Usuário padrão do bairro.

#### Permissões:

- visualizar conteúdo do bairro
- criar posts de aviso
- criar posts de negócio
- interagir com status (confirmar eventos)

#### Restrições:

- não pode aprovar membros
- não pode remover usuários
- não pode remover posts de terceiros
- não pode alterar roles

---

### 3.2 Admin

Usuário com permissões de moderação.

#### Permissões:

- todas as permissões de membro
- aprovar ou rejeitar solicitações de entrada
- remover membros
- remover posts de qualquer usuário
- promover membro para admin

#### Restrições:

- não pode remover outros admins
- não pode remover o superadmin
- não pode promover admin para superadmin
- não pode deletar o bairro

---

### 3.3 Superadmin

Usuário com controle total sobre o bairro.

#### Permissões:

- todas as permissões de admin
- promover e rebaixar admins
- remover admins
- promover admin para superadmin
- deletar o bairro
- transferir propriedade do bairro (superadmin)

---

## 4. Regras de Alteração de Roles

### Promoções

- membro pode ser promovido para admin por admin ou superadmin
- admin pode ser promovido para superadmin apenas pelo superadmin atual

---

### Rebaixamentos

- admin pode ser rebaixado para membro apenas pelo superadmin
- superadmin não pode ser rebaixado diretamente

---

## 5. Transferência de Superadmin

A transferência de superadmin deve seguir as seguintes regras:

- apenas o superadmin atual pode iniciar a transferência
- o novo superadmin deve ser um usuário já pertencente ao bairro
- recomenda-se que o usuário já seja admin (opcional)

### Fluxo:

1. superadmin seleciona um usuário elegível
2. sistema solicita confirmação
3. após confirmação:
   - o usuário selecionado se torna superadmin
   - o superadmin anterior passa a ser admin

---

## 6. Regras de Saída do Bairro

### Membro e Admin

- podem sair do bairro a qualquer momento
- ao sair, perdem acesso imediatamente

---

### Superadmin

- não pode sair do bairro sem transferir sua função
- o sistema deve bloquear a saída caso não exista outro superadmin definido

---

## 7. Regras de Consistência

O sistema deve garantir:

- sempre existir exatamente um superadmin por bairro
- nenhuma ação pode violar a hierarquia de permissões
- alterações de role devem ser validadas antes de serem aplicadas

---

## 8. Representação (para implementação)

Sugestão de valores:

- membro
- admin
- superadmin

Exemplo:

```ts
type Role = "membro" | "admin" | "superadmin";
```

---

## 9. Validação de Permissões (Exemplo)

Exemplo de regra:

if (user.role === "admin") {
  // pode aprovar membros
}

if (user.role === "superadmin") {
  // pode deletar bairro
}

---

## 10. Observações

Todas as verificações de permissão devem ser feitas no backend
O frontend deve apenas refletir permissões (exibir ou ocultar ações)
Nenhuma ação crítica deve depender apenas do frontend