#  Guia de Implantação do Banco de Dados: Ciência é Fundamental (Supabase)

Este repositório documenta a inicialização, configuração de segurança e execução do schema relacional do projeto **Ciência é Fundamental** utilizando a infraestrutura do **Supabase (PostgreSQL 15+)**.

---

## Visão Geral do Domínio e Escopo

O ecossistema é centrado no conceito unificado de **E-book** (que engloba a visualização de material de leitura ou plano de aula estruturado).
- **Tipos de Usuário:** `ADMIN` (cria atividades oficiais e gerencia a plataforma) e `PROFESSOR` (cria seus próprios e-books privados a partir de sugestões de atividades).
- **Relacionamentos Centrais:** 
  - Estrutura de domínio normalizada (`estado`, `cidade`, `disciplina`, `serie`).
  - Atividades com mapeamento de materiais necessários e substitutos (`material_alternativo`).
  - Associação $N:M$ entre e-books e atividades com controle de checklist, ordem de execução e anotações/devolutivas pedagógicas.
  - Conformidade LGPD com consentimento rastreável por versão, data e IP.

---

## 🛠️ Criação do Projeto no Supabase

1. Acesse o console oficial: [Supabase Dashboard](https://supabase.com/dashboard).
2. Clique em **"New Project"** e selecione a sua organização.
3. Preencha as configurações iniciais:
   - **Name:** `ciencia-fundamental` (ou nome de sua preferência).
   - **Database Password:** Gere uma senha forte e guarde-a em um gerenciador seguro.
   - **Region:** Selecione a região mais próxima do público-alvo (ex: `sa-east-1` - São Paulo) para reduzir a latência de rede.
   - **Pricing Plan:** Free / Pro (conforme sua cota).

---

## Diretrizes de Segurança do Supabase

Durante ou logo após a inicialização do projeto e ao gerenciar o schema, o Supabase disponibiliza opções cruciais de segurança para a camada de dados. Abaixo estão as recomendações técnicas e o embasamento arquitetural de cada uma:

### 1. `Enable Data API`
> *Autogenerate a RESTful API for your public schema. Recommended if using a client library like supabase-js.*

* **Decisão:** **HABILITAR** (se for consumir via cliente frontend/mobile ou Supabase Studio) **OU MANTER EM CONFORMIDADE COM O BACKEND DJANGO**.
* **Motivo:** O PostgREST cria endpoints RESTful instantâneos mapeados sobre as tabelas do schema `public`. No entanto, como a arquitetura do projeto possui um **backend Django centralizado**, a Data API deve ser usada com cautela: se o Django for o único consumidor direto do banco via pooler/conexão direta (libpq/psycopg), a Data API pode até ser desativada futuramente para eliminar superfícies de ataque públicas. Caso utilize o painel interativo do Supabase ou microsserviços frontend via `supabase-js`, mantenha-a ativada.

---

### 2. `Automatically expose new tables`
> *Grants privileges to Data API roles by default, exposing new tables. We recommend disabling this to control access manually.*

* **Decisão:** **DESABILITAR (Disable / Off)**.
* **Motivo Técnico (Princípio do Menor Privilégio):**
  - Por padrão, quando ativado, concede permissões de leitura/escrita para as roles anônimas (`anon`) e autenticadas (`authenticated`) do PostgREST assim que um comando `CREATE TABLE` é executado.
  - Como o schema contém tabelas altamente sensíveis (como `usuario`, com hashes de senha, dados de auditoria LGPD e CPF), **novas tabelas NUNCA devem ser expostas publicamente por padrão**.
  - O controle manual garante que apenas as tabelas estritamente necessárias tenham permissão concedida explicitamente (`GRANT SELECT ON ... TO anon/authenticated`).

---

### 3. `Enable automatic RLS` (Row Level Security)
> *Create an event trigger that automatically enables Row Level Security on all new tables in the public schema.*

* **Decisão:** **HABILITAR (Enable / On)**.
* **Motivo Técnico (Defesa em Profundidade):**
  - O **Row Level Security (RLS)** é o mecanismo do PostgreSQL que avalia se uma role tem permissão para enxergar ou manipular linhas individuais.
  - Ao ativar o RLS por padrão via gatilho de evento, qualquer tabela recém-criada fica **bloqueada para acesso anônimo/externo por padrão** (`Default Deny`), prevenindo vazamentos acidentais de dados antes que políticas (`CREATE POLICY`) específicas de segurança sejam criadas.
  - Se o backend for exclusivamente um ORM Django conectando-se como superusuário/usuário de serviço (`postgres`), ele ignora o RLS (ou tem `BYPASSRLS`), permitindo que a aplicação funcione perfeitamente enquanto bloqueia acessos indevidos via PostgREST direto.

---
