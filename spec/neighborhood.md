# Neighborhood (Bairro) Specification

## 1. Visão Geral

Um bairro representa uma comunidade de usuários dentro do sistema.

Todas as interações do usuário (avisos, negócios e status) estão associadas a um único bairro.

---

## 2. Regras Gerais

- Um usuário pode estar vinculado a apenas um bairro por vez
- Um bairro pode conter múltiplos usuários
- Cada bairro deve possuir exatamente um superadmin
- Usuários só podem visualizar e interagir com conteúdos do bairro ao qual pertencem

---

## 3. Criação de Bairro

### 3.1 Condições

- O usuário deve estar autenticado
- O usuário não pode já estar vinculado a outro bairro

---

### 3.2 Processo de Criação

Ao criar um bairro, o sistema deve:

1. Registrar os dados do bairro
2. Associar o usuário criador ao bairro
3. Definir o usuário como **superadmin**
4. Gerar um código único de acesso
5. Gerar um link de convite

---

### 3.3 Dados do Bairro

Campos mínimos:

- nome do bairro (string)
- cidade (string)
- estado (string)
- código de acesso (string único)
- link de convite (string único)
- data de criação

---

### 3.4 Observações

- O nome do bairro é definido pelo usuário e não precisa ser validado externamente
- Cidade e estado podem ser obtidos via API externa (ex: BrasilAPI)

---

## 4. Entrada em um Bairro

O sistema deve permitir entrada em um bairro por dois métodos:

---

### 4.1 Entrada via Link de Convite

#### Regras:

- O link deve ser único por bairro
- O link pode ser reutilizável

#### Fluxo:

1. Usuário acessa o link
2. Caso não esteja autenticado:
   - deve realizar login ou cadastro
3. Após autenticação:
   - usuário é automaticamente vinculado ao bairro

---

### 4.2 Entrada via Código de Acesso

#### Regras:

- O código deve identificar unicamente o bairro
- Entrada requer aprovação

#### Fluxo:

1. Usuário informa o código
2. Sistema cria uma solicitação de entrada com status "pendente"
3. Admin ou superadmin aprova ou rejeita a solicitação
4. Após aprovação:
   - usuário é vinculado ao bairro

---

## 5. Estados de Associação

Um usuário em relação a um bairro pode estar em um dos seguintes estados:

- não vinculado
- pendente (aguardando aprovação)
- ativo (membro do bairro)

---

## 6. Convites e Acesso

### Link de Convite

- permite entrada imediata
- não requer aprovação
- pode ser compartilhado livremente

---

### Código de Acesso

- requer aprovação manual
- utilizado para controle mais restrito

---

## 7. Saída do Bairro

### 7.1 Membro e Admin

- podem sair do bairro a qualquer momento
- ao sair:
  - perdem acesso ao conteúdo
  - deixam de estar associados ao bairro

---

### 7.2 Superadmin

- não pode sair sem transferir a função
- deve transferir o papel de superadmin antes de sair

---

## 8. Exclusão de Bairro

### Permissão

- apenas o superadmin pode deletar o bairro

---

### Comportamento

Ao deletar um bairro:

- todos os vínculos com usuários são removidos
- todos os conteúdos associados (avisos, negócios, status) são removidos

---

## 9. Regras de Consistência

O sistema deve garantir:

- um usuário não pode pertencer a mais de um bairro simultaneamente
- um bairro sempre deve ter exatamente um superadmin
- não pode existir bairro sem usuários
- todas as ações devem validar associação do usuário ao bairro

---

## 10. Identificação do Bairro

Sugestão de identificadores:

- id (interno, UUID ou incremental)
- código de acesso (curto e compartilhável)
- link de convite (URL)

---

## 11. Observações de Implementação

- validações de associação devem ocorrer no backend
- o frontend deve apenas refletir o estado (ativo, pendente, etc.)
- o código de acesso deve ser único e fácil de compartilhar
- o link de convite pode conter o identificador do bairro (ex: token)
