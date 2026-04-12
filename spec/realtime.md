---
tags: [spec, realtime, status, confirmations, social-proof]
aliases: [Real-time Status, Status do Bairro, Realtime Spec]
---

# Realtime Status Specification

## 1. Visão Geral

O sistema de Status do Bairro permite que usuários reportem e confirmem eventos em tempo real de forma rápida e sem necessidade de criação de posts.

Diferente dos avisos, o status não possui descrição detalhada e é baseado em interações simples (cliques), com foco em velocidade e validação coletiva.

---

## 2. Objetivo

- Reduzir a necessidade de criação de múltiplos posts para o mesmo evento
- Permitir identificação rápida de problemas no bairro
- Fornecer validação social através de confirmações de múltiplos usuários

---

## 3. Tipos de Status

O sistema deve suportar os seguintes tipos fixos:

- falta_agua
- falta_energia
- barulho_excessivo
- coleta_lixo

---

## 4. Estrutura de Status

Campos principais:

- id
- tipo
- bairro_id
- data_inicio
- data_expiracao
- total_confirmacoes

---

## 5. Confirmações (Votos)

Cada usuário pode interagir com um status através de uma confirmação.

### Regras:

- um usuário pode confirmar apenas uma vez por tipo de status ativo
- a confirmação incrementa o contador total
- o usuário pode remover sua confirmação (opcional)

---

## 6. Comportamento

### Criação automática

- Um status é criado automaticamente quando o primeiro usuário confirma um tipo específico

---

### Atualização

- Novas confirmações aumentam o contador
- O status permanece ativo enquanto estiver dentro do tempo de validade

---

### Expiração

- Cada status possui um tempo de vida curto (5 horas — definido por `STATUS_DURATION_HOURS = 5`)
- Após expiração:
  - o status é desativado
  - contadores são resetados
  - novas confirmações criam um novo status

---

## 7. Prova Social

O sistema deve exibir a quantidade de confirmações:

Exemplo:
- "5 pessoas confirmaram falta de água"

Objetivo:
- validar a informação
- evitar múltiplos avisos duplicados

---

## 8. Relação com Avisos

### Regra de integração (opcional)

- Se um status atingir um número mínimo de confirmações (ex: 5):
  - o sistema pode gerar automaticamente um aviso

Exemplo:
- "Diversos moradores reportaram falta de água"

---

## 9. Permissões

- membro: pode confirmar status
- admin: pode confirmar status
- superadmin: pode confirmar status

Nenhum usuário cria status manualmente.

---

## 10. Visualização

Os status devem ser exibidos:

- na tela principal (Status do Bairro)
- em formato de ícones com indicadores visuais
- com destaque para os mais ativos

---

## 11. Regras de Consistência

- só pode existir um status ativo por tipo e por bairro
- confirmações devem estar vinculadas a usuários
- usuários devem pertencer ao bairro para interagir

---

## 12. Observações de Implementação

- recomenda-se armazenar confirmações em uma tabela separada

Exemplo:

status_confirmations:
- id
- status_id
- user_id

---

- o total de confirmações pode ser:
  - calculado dinamicamente
  - ou armazenado para performance

---

## 13. Possíveis Evoluções Futuras

- adicionar novos tipos de status
- permitir comentários vinculados ao status
- exibir histórico de eventos
- integração com notificações push