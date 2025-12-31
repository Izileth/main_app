# Documentação do Banco de Dados e Regras de Negócio

Este documento detalha a estrutura do banco de dados, os relacionamentos entre as tabelas e as regras de negócio da aplicação, com base nas migrações do Supabase.

## Visão Geral

O banco de dados é projetado para suportar uma plataforma social com perfis de usuário, posts, sistema de seguidores, categorias, tags e interações como curtidas, descurtidas, comentários e visualizações. As regras de negócio são fortemente aplicadas no nível do banco de dados usando Row-Level Security (RLS), Triggers e Funções PostgreSQL.

---

## Extensões Ativadas

- `unaccent`: Usada para remover acentos de textos, principalmente para a geração de slugs.

---

## Tabelas e Entidades

### 1. `profiles`

Armazena os dados públicos e privados dos perfis de usuário.

**Estrutura da Tabela:**

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `id` | `uuid` | Chave primária, referenciando `auth.users.id`. |
| `name` | `text` | Nome completo do usuário. |
| `first_name` | `text` | Primeiro nome do usuário (vindo dos metadados). |
| `last_name` | `text` | Último nome do usuário (vindo dos metadados). |
| `avatar_url` | `text` | URL da imagem de avatar do usuário. |
| `banner_url` | `text` | URL da imagem de banner do usuário. |
| `bio` | `text` | Biografia do usuário. |
| `role` | `text` | Papel do usuário (`ADM` ou `US`). Padrão: `US`. |
| `position` | `text` | Cargo ou posição do usuário. |
| `social_media_links`| `jsonb` | Objeto JSON com links para redes sociais. |
| `website` | `text` | URL do site pessoal do usuário. |
| `location` | `text` | Localização do usuário. |
| `birth_date` | `date` | Data de nascimento do usuário. |
| `slug` | `text` | Slug único e não nulo para a URL do perfil. |
| `email` | `text` | Email único e não nulo do usuário. |
| `password` | `text` | Campo para senha (uso a ser definido). |
| `updated_at` | `timestamptz`| Data da última atualização. |
| `created_at` | `timestamptz`| Data de criação. |

**Regras de Negócio (RLS & Triggers):**

- **Visualização:** Perfis públicos podem ser visualizados por qualquer pessoa.
- **Criação:** Usuários só podem criar seu próprio perfil.
- **Atualização:** Usuários só podem atualizar seu próprio perfil.
- **Gatilho `on_auth_user_created`:**
  - **Ação:** Quando um novo usuário é criado em `auth.users`, um perfil correspondente é criado automaticamente na tabela `profiles`.
  - **Lógica:** A função `handle_new_user()` é acionada. Ela extrai `first_name` e `last_name` dos metadados do usuário para formar um nome completo, gera um slug único com a função `generate_profile_slug()`, e insere o novo perfil com os dados básicos. Inclui tratamento de exceções para depuração.
- **Gatilho `on_profile_updated`:**
  - **Ação:** Quando um perfil na tabela `profiles` é atualizado.
  - **Lógica:** A função `handle_profile_update()` é acionada. Se o `slug`, `name` ou `avatar_url` do perfil mudou, ela atualiza todos os posts (`posts`) feitos por esse usuário para refletir os novos dados (denormalização).

---

### 2. `posts`

Armazena o conteúdo principal criado pelos usuários.

**Estrutura da Tabela:**

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `id` | `bigint` | Chave primária autoincrementada. |
| `user_id` | `uuid` | Chave estrangeira para `profiles.id`. Define o autor. |
| `title` | `text` | Título do post. |
| `content` | `text` | Conteúdo principal do post em formato de texto. |
| `description` | `text` | Uma breve descrição ou resumo do post. |
| `slug` | `text` | Slug único para a URL do post. |
| `likes_count` | `integer` | Contagem de curtidas. Padrão: 0. |
| `dislikes_count` | `integer` | Contagem de descurtidas. Padrão: 0. |
| `views_count` | `integer` | Contagem de visualizações. Padrão: 0. |
| `comments_count` | `integer` | Contagem de comentários. Padrão: 0. |
| `profile_slug` | `text` | Slug do autor (denormalizado para performance). |
| `profile_name` | `text` | Nome do autor (denormalizado para performance). |
| `profile_avatar_url`| `text` | URL do avatar do autor (denormalizado). |
| `updated_at` | `timestamptz`| Data da última atualização. |
| `created_at` | `timestamptz`| Data de criação. |

**Regras de Negócio (RLS & Triggers):**

- **Visualização:** Posts podem ser visualizados por qualquer pessoa.
- **Criação:** Usuários autenticados podem criar posts para si mesmos.
- **Atualização:** Usuários só podem atualizar seus próprios posts.
- **Exclusão:** Usuários só podem excluir seus próprios posts.
- **Gatilhos de Contagem:** As colunas `likes_count`, `dislikes_count` e `comments_count` são atualizadas automaticamente por gatilhos nas tabelas `likes`, `dislikes` e `comments`, respectivamente. A `views_count` é atualizada por um gatilho na tabela `post_views`.

---

### 3. `followers`

Tabela de associação para o sistema de "seguir".

**Estrutura da Tabela:**

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `follower_id` | `uuid` | ID do perfil que está seguindo. Parte da PK. |
| `following_id` | `uuid` | ID do perfil que está sendo seguido. Parte da PK. |
| `created_at` | `timestamptz`| Data da criação da relação. |

**Regras de Negócio (RLS):**

- **Visualização:** Qualquer pessoa pode ver as relações de "seguir".
- **Criação:** Usuários autenticados podem seguir outros perfis (inserir em seu próprio nome).
- **Exclusão:** Usuários podem deixar de seguir outros (excluir relações que eles criaram).

---

### 4. `categories` & `post_categories`

Sistema de categorização para os posts.

**`categories` (Estrutura):**

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `id` | `bigint` | Chave primária autoincrementada. |
| `name` | `text` | Nome único da categoria. |
| `created_at` | `timestamptz`| Data de criação. |

**`post_categories` (Tabela de Junção):**

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `post_id` | `bigint` | Referência ao `posts.id`. |
| `category_id` | `bigint` | Referência ao `categories.id`. |

**Regras de Negócio (RLS):**

- **`categories`:**
  - **Visualização:** Categorias são visíveis para todos.
  - **Criação:** Qualquer usuário autenticado pode criar novas categorias.
- **`post_categories`:**
  - **Visualização:** Relações são visíveis para todos.
  - **Criação/Exclusão:** Apenas o autor do post pode adicionar ou remover categorias de seu post.

---

### 5. `tags` & `post_tags`

Sistema de tags para os posts.

**`tags` (Estrutura):**

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `id` | `bigint` | Chave primária autoincrementada. |
| `name` | `text` | Nome único da tag. |
| `created_at` | `timestamptz`| Data de criação. |

**`post_tags` (Tabela de Junção):**

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `post_id` | `bigint` | Referência ao `posts.id`. |
| `tag_id` | `bigint` | Referência ao `tags.id`. |

**Regras de Negócio (RLS):**

- **`tags`:**
  - **Visualização:** Tags são visíveis para todos.
  - **Criação:** Qualquer usuário autenticado pode criar novas tags.
- **`post_tags`:**
  - **Visualização:** Relações são visíveis para todos.
  - **Criação/Exclusão:** Apenas o autor do post pode adicionar ou remover tags de seu post.

---

### 6. `likes`, `dislikes`, `comments`, `post_views` (Tabelas de Engajamento)

**Estrutura Comum:**

- **`likes`**: `user_id`, `post_id`, `created_at`
- **`dislikes`**: `user_id`, `post_id`, `created_at`
- **`post_views`**: `post_id`, `user_id`, `created_at` (Chave primária em `(post_id, user_id)`)
- **`comments`**: `id`, `post_id`, `user_id`, `content`, `parent_comment_id`, `created_at`, `updated_at`

**Regras de Negócio (RLS & Triggers):**

- **Visualização:** Todos podem ver todos os registros (exceto `post_views`, onde o usuário só vê suas próprias visualizações).
- **Criação:** Usuários autenticados podem criar registros em seu próprio nome.
- **Exclusão/Atualização:** Usuários só podem modificar ou excluir seus próprios registros.
- **Gatilhos:**
  - A inserção/exclusão em `likes` incrementa/decrementa `posts.likes_count`.
  - A inserção/exclusão em `dislikes` incrementa/decrementa `posts.dislikes_count`.
  - A inserção/exclusão em `comments` incrementa/decrementa `posts.comments_count`.
  - A inserção em `post_views` incrementa `posts.views_count`. A chave primária `(post_id, user_id)` garante que um usuário só conte como uma visualização por post.

---

### 7. `post_images`

Armazena as imagens associadas a um post, permitindo múltiplas imagens por post.

**Estrutura da Tabela:**

| Coluna | Tipo | Descrição |
| :--- | :--- | :--- |
| `id` | `bigint` | Chave primária autoincrementada. |
| `post_id` | `bigint` | Referência ao `posts.id`. |
| `image_url` | `text` | URL da imagem. |
| `created_at` | `timestamptz`| Data de criação. |

**Regras de Negócio (RLS):**

- **Visualização:** Imagens de posts são visíveis para todos.
- **Criação/Exclusão:** Apenas o autor do post pode adicionar ou remover imagens de seu post.

---

## Funções de Apoio (Helper Functions)

- **`slugify(value text)`:**
  - **Propósito:** Converte uma string em um formato de slug (URL-friendly).
  - **Lógica:** Remove acentos, converte para minúsculas, substitui espaços por hífens e remove caracteres não alfanuméricos.

- **`generate_profile_slug(name text)`:**
  - **Propósito:** Gera um slug único para um perfil.
  - **Lógica:** Usa `slugify()` para criar um slug base. Se o slug já existir na tabela `profiles`, anexa um sufixo numérico (ex: `meu-nome-2`) até encontrar um valor único. Contém uma lógica de fallback para `user` se o nome for nulo.

---

## Storage (Buckets)

Três buckets públicos são configurados para armazenar arquivos.

- **`avatars`:**
  - **Propósito:** Armazenar imagens de avatar dos usuários.
  - **Políticas:** Acessível publicamente para leitura. Qualquer pessoa pode fazer upload.
- **`banners`:**
  - **Propósito:** Armazenar imagens de banner dos perfis.
  - **Políticas:** Acessível publicamente para leitura. Qualquer pessoa pode fazer upload.
- **`posts`:**
  - **Propósito:** Armazenar imagens associadas aos posts.
  - **Políticas:** Acessível publicamente para leitura. Qualquer pessoa pode fazer upload.
