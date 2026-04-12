---
tags: [spec, product, vision, mvp]
aliases: [Product Vision, Product Spec]
---

# Product Specification — Meu Bairro

## 1. Visão Geral

Meu Bairro é um aplicativo mobile que conecta moradores de uma mesma região (bairro), permitindo comunicação organizada, troca de serviços e compartilhamento de informações em tempo real.

O objetivo principal é substituir a comunicação desorganizada de grupos de WhatsApp por uma plataforma estruturada, com separação clara de contexto e informações relevantes.

---

## 2. Problema

Atualmente, a comunicação entre vizinhos ocorre majoritariamente via grupos de mensagens, o que gera:

- Alto volume de mensagens irrelevantes
- Perda de informações importantes
- Mistura de contextos (vendas, avisos, eventos)
- Dificuldade de encontrar informações úteis
- Falta de validação coletiva (ex: várias pessoas perguntando a mesma coisa)

---

## 3. Solução

O sistema propõe uma divisão clara da comunicação em três áreas principais:

- Status do Bairro (tempo real)
- Avisos (comunicação estruturada)
- Negócios (marketplace local)

Cada área possui regras próprias, evitando poluição de informação e melhorando a experiência do usuário.

---

## 4. Público-Alvo

- Moradores de bairros
- Comunidades locais
- Pequenos prestadores de serviço
- Pessoas interessadas em comunicação local organizada

---

## 5. Funcionalidades Principais

### 5.1 Status do Bairro

- Permite que usuários confirmem eventos em tempo real
- Baseado em ações rápidas (sem criação de post)
- Exibe prova social (quantidade de confirmações)
- Eventos possuem expiração automática

Exemplos:
- Falta de água
- Falta de energia
- Barulho excessivo
- Coleta de lixo

---

### 5.2 Avisos

- Permite criação de posts estruturados
- Contém descrição, categoria e contexto
- Possui tempo de expiração automático

Categorias:
- Segurança
- Utilidade Pública
- Eventos

---

### 5.3 Negócios

- Permite anúncios locais (produtos e serviços)
- Comunicação externa via WhatsApp
- Não possui sistema de chat interno
- Posts possuem expiração automática

---

## 6. Regras de Negócio Gerais

- Um usuário pode participar de apenas um bairro por vez
- Cada bairro possui exatamente um Superadmin
- Usuários podem ter roles: membro, admin ou superadmin
- Conteúdos possuem tempo de expiração automático
- Apenas membros do bairro podem interagir com conteúdo

---

## 7. Diferenciais do Produto

- Separação clara entre tipos de conteúdo
- Sistema de expiração automática de posts
- Status em tempo real com validação coletiva
- Simplicidade de uso (foco em ações rápidas)
- Integração direta com WhatsApp para comunicação externa

---

## 8. Plataforma

- Aplicativo mobile (React Native com Expo)
- Backend baseado em API REST
- Banco de dados relacional

---

## 9. Objetivo do MVP

Entregar uma versão funcional que permita:

- Cadastro e autenticação de usuários
- Criação e entrada em bairros
- Criação e visualização de avisos
- Criação e interação com anúncios
- Uso do sistema de status em tempo real

---

## 10. Fora do Escopo 

- Chat interno entre usuários
- Sistema de pagamento
- Geolocalização avançada
- Múltiplos bairros por usuário
- Sistema de reputação
