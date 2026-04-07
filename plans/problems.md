# Plano de Refatoração e Correções

Este plano detalha como endereçaremos os 7 pontos levantados sobre validações faltantes, duplicação de código, performance e melhorias no schema.

## User Review Required

> [!IMPORTANT]
> Aprovação Necessária
> Por favor, revise o plano abaixo. As alterações propostas ajustam regras de negócio importantes (como não poder promover a si próprio e restrições de convite). Nenhuma alteração disruptiva está prevista.

> [!WARNING]
> Migração do Banco de Dados
> A alteração do tamanho do campo `contact` na tabela `posts` exigirá a geração e execução de uma migração (`drizzle-kit generate` e `drizzle-kit push`).

## Proposed Changes

---

### Serviço de Membros (member.service.ts)

#### [MODIFY] member.service.ts
**Problema:** O bloco que impede o usuário verificar a si próprio estava vazio e confuso com `split('|')`.
**Solução:** Implementar a verificação utilizando diretamente o `actor.id` (que neste contexto é o ID do membro fornecido na chamada).
- **Código:**
  ```typescript
  // Cannot change own role
  if (target.id === actor.id) {
    throw new AppError('Não pode alterar próprio cargo', 403, 'FORBIDDEN');
  }
  ```

---

### Serviço de Bairros (neighborhood.service.ts)

#### [MODIFY] neighborhood.service.ts
**Problema:** `joinByLink` permite que um membro com `deleted_at` assuma que o vínculo já existe bloqueando um novo join, ou vice-versa (não prevê membros deletados).
**Solução:** Adicionar a condição `isNull(neighborhoodMembers.deleted_at)` no passo de verificação de `existing` para garantir a consistência com o resto do sistema.

---

### Serviço de Posts (post.service.ts)

#### [MODIFY] post.service.ts
**Problema:** Duplicação de código e filtragem de resultados em memória (incluindo a lógica não finalizada do `$dynamic()`).
**Solução:** 
1. **Extrair deleção de posts:** Criar o método privado `deletePost` e reutilizá-lo em `deleteNotice` e `deleteBusiness`.
2. **Performance (Filtro de Expiração no Banco):** Fazer as queries de `listNotices` e `listBusiness` utilizarem `gt(posts.expires_at, now)` dentro do banco de dados em vez de filtrar em JS.
3. **Uso de `$dynamic()` e Filtro de Categoria:** Substituir a construção dinâmica e o array `.filter` por um array de condições (`filters`) passado para o bloco `where(and(...filters))`, permitindo adicionar o filtro de categoria de forma orgânica e em SQL.

---

### Constantes e Configurações

#### [NEW] config/constants.ts
**Problema:** Valores mágicos espalhados.
**Solução:** Criar `src/config/constants.ts` (ou adicionar ao `utils/constants.ts` existente) para centralizar a duração do auth e status.
- **Conteúdo:**
  ```typescript
  export const TOKEN_EXPIRY = {
    ACCESS_DAYS: 7,
    REFRESH_DAYS: 30,
    REFRESH_MS: 30 * 24 * 60 * 60 * 1000,
  } as const;

  export const STATUS_DURATION_HOURS = 5;
  ```

#### [MODIFY] auth.service.ts
- Substituir o multiplicador em hardcode de 30 dias pela nova constante `TOKEN_EXPIRY.REFRESH_MS`.

#### [MODIFY] status.service.ts
- Importar a constante `STATUS_DURATION_HOURS` do novo arquivo (removendo a declaração redundante no topo).

---

### Banco de Dados (schema.ts)

#### [MODIFY] schema.ts
**Problema:** Tamanho do `varchar` de `contact` inadequado (apenas 20 caracteres).
**Solução:** 
- Aumentar o `length` de 20 para 30 em `posts.contact`.
- **Comando Adicional:** Executar a migração via Drizzle Kit na fase de execução.

## Open Questions

Não há dúvidas referentes à viabilidade. Se estiver de acordo, execute o comando de aprovação e iniciaremos a implementação desses 7 pontos!

## Verification Plan

### Automated / Dev Tests
- Iniciar o servidor com `npm run dev` para garantir que as rotas de auth continuem funcionando.
- Validar se o Drizzle efetuou o push das alterações do tamanho da coluna perfeitamente.
- Testar visualmente ou via API uma busca a um post expirado para atestar que o banco o esconde.
