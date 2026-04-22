# Curso de SQLAlchemy

Este repositório contém o material e exercícios para o curso de SQLAlchemy. O objetivo é aprender a usar o `SQLAlchemy`, uma biblioteca Python para trabalhar com bancos de dados relacionais.

## Contexto do Curso

O curso é estruturado em torno da empresa fictícia **DataVendas**, um e-commerce cujo banco de dados você vai construir do zero ao longo das aulas. A estrutura de dados trabalhada é:

```
clientes  ──<  pedidos  ──<  itens_pedidos  >──  produtos
```

---

## Conteúdo das Aulas

### Setup do Projeto & Conexão com o Banco (`01_setup_e_conexao.ipynb`)

Introdução ao SQLAlchemy e configuração do ambiente de desenvolvimento.

**O que você vai aprender:**
- O que é o SQLAlchemy e por que ele existe
- Como criar a `Engine` — o ponto de entrada para o banco
- Formato da URL de conexão para SQLite e PostgreSQL
- Como testar a conexão com `engine.connect()` e o padrão `with`
- As duas camadas do SQLAlchemy: Core e ORM
- Como inspecionar o banco com `inspect(engine)`
- Boas práticas: carregar credenciais via variáveis de ambiente com `python-dotenv`

---

### Modelando o Banco com SQLAlchemy Core (`02_modelando_com_core.ipynb`)

Criação do schema do banco via código Python, com controle de versão pelo Git.

**O que você vai aprender:**
- Usar `MetaData` como catálogo central das tabelas
- Criar tabelas com `Table` e `Column`
- Definir tipos de dados: `Integer`, `String`, `DateTime`, `Numeric`, `Boolean`
- Aplicar restrições de integridade: `primary_key`, `nullable`, `unique`
- Relacionar tabelas com `ForeignKey`
- Garantir integridade de domínio com `CheckConstraint`
- Materializar o schema no banco com `metadata.create_all(engine)`
- Inspecionar tabelas e foreign keys com `Inspector`


**Tabelas criadas:**
- `tb_clientes`, `tb_produtos`, `tb_pedidos`, `tb_itens_pedidos`

---

### Trabalhando com ORM: Modelos e Sessão (`03_orm_modelos_e_sessao.ipynb`)

Trabalho com o banco usando classes Python em vez de strings SQL.

**O que você vai aprender:**
- Criar modelos com `DeclarativeBase` e herança de `Base`
- Definir colunas com `Mapped[tipo]` e `mapped_column(...)`
- Criar relacionamentos de navegação com `relationship(...)`
- Usar a `Session` para operações CRUD
- Entender transações: `commit()`, `rollback()` e `flush()`
- Popular o banco com dados sintéticos usando `Faker`

**Modelos criados:** `Cliente`, `Produto`, `Pedido`, `ItemPedido`

---

### Segurança: SQL Injection (`04_sql_injection_seguranca.ipynb`)

Como proteger seu código contra a vulnerabilidade mais comum em sistemas com banco de dados.

**O que você vai aprender:**
- O que é SQL Injection e por que é perigoso
- Ver o ataque acontecendo (de forma controlada)
- Por que o SQLAlchemy sozinho não resolve o problema
- Prevenir com consultas parametrizadas
- Adicionar validação de tipo como camada extra de defesa

---

### Consultando Dados: select(), Filtros e Paginação (`05_consultando_dados.ipynb`)

Traduzindo perguntas de negócio em queries Python com o `select()` moderno do SQLAlchemy 2.x.

**O que você vai aprender:**
- Usar `select()` para montar consultas (lazy — só vai ao banco no `execute()`)
- Escolher o formato de retorno certo para cada situação: `scalars()`, `execute()` e `mappings()`
- Filtrar com `where()`, `and_()`, `or_()` e `like()`
- Ordenar resultados com `order_by()` e `.desc()` / `.asc()`
- Paginar resultados com `limit()` e `offset()`
- Construir queries dinâmicas em etapas
---

### Relacionamentos, N+1 e Entregando Dados para o Pandas (`06_relacionamentos_n1_pandas.ipynb`)

Diagnóstico e correção do problema N+1, e pipeline para entrega de dados ao Pandas.

**O que você vai aprender:**
- O que é lazy loading e por que é desastroso em loops
- Identificar o problema N+1 observando o log de queries (`echo=True`)
- Corrigir com `joinedload` (usa JOIN) e `selectinload` (usa `WHERE IN`)
- Converter resultados diretamente para DataFrames do Pandas

---

### Consultas Compostas: UNION, Subconsultas, IN e EXISTS (`07_consultas_compostas.ipynb`)

Técnicas para responder perguntas que uma query simples não consegue dar.

**O que você vai aprender:**
- `UNION` / `UNION ALL` para combinar conjuntos de resultados
- `.subquery()` para usar uma query como etapa intermediária de outra
- `.in_()` / `.notin_()` para filtrar por pertencimento a uma lista ou subquery
- `exists()` / `~exists()` para filtrar pela existência de registros relacionados
- Construir queries progressivamente em etapas (o `select()` só vai ao banco no `execute()`)

---

### Filtrando Grupos: HAVING e ANY (`08_any_e_having.ipynb`)

Filtros sobre resultados agregados e comparações com conjuntos.

**O que você vai aprender:**
- `HAVING` para filtrar grupos depois do `GROUP BY`
- Combinar `WHERE` e `HAVING` na mesma query
- `ANY` para comparar um valor com ao menos um elemento de uma subconsulta
- Alternativa ao `ANY` usando `func.min()` com `.scalar_subquery()`

---

### JOINs: INNER, LEFT, Múltiplos e Self-Join (`09_joins_avancados.ipynb`)

Domínio completo de JOINs, incluindo diagnóstico de bugs silenciosos.

**O que você vai aprender:**
- A diferença prática entre INNER JOIN e LEFT OUTER JOIN
- Como um `JOIN` onde deveria ser `LEFT JOIN` pode excluir registros silenciosamente
- JOINs via relacionamento ORM vs JOINs explícitos com condição manual
- JOINs múltiplos encadeados (cliente → pedido → item → produto)
- Self-join com `aliased()`: comparar a tabela com ela mesma
- JOIN + GROUP BY para métricas por cliente
- Inspecionar o SQL gerado com `stmt.compile(dialect=sqlite.dialect(), compile_kwargs={"literal_binds": True})`

---

### Views e Triggers: Lógica que Vive no Banco (`10_views_e_triggers.ipynb`)

Encapsulamento de JOINs complexos em views e manutenção automática de consistência com triggers.

**O que você vai aprender:**
- Criar views com `text()` e `engine.begin()` para encapsular JOINs reutilizáveis
- Consultar views de três formas: SQL nativo, `Table` refletida e Pandas
- Criar triggers `AFTER INSERT`, `AFTER UPDATE` e `AFTER DELETE`
- Usar `session.refresh()` para sincronizar objetos Python com mudanças feitas pelo banco
- Inspecionar views e triggers existentes via `sqlite_master`

**Views criadas:**
- `vw_vendas_detalhadas` — 1 linha por item (análise de produto)
- `vw_pedidos_resumo` — 1 linha por pedido (métricas de receita)

---

### UPDATE, DELETE, Transações (`11_update_delete_transacoes_core.ipynb`)

Modificação segura de dados, transações atômicas e operações de alto volume.

**O que você vai aprender:**
- Atualizar registros via ORM (poucos registros, lógica Python) ou via `update()` direto (alto volume)
- Usar `bulk_update_mappings()` para atualizar lotes com dicionários
- Apagar registros respeitando integridade referencial (`PRAGMA foreign_keys=ON` no SQLite)
- Usar transações para garantir operações "tudo ou nada" com `with session.begin()`
- Diagnosticar e testar rollback automático em falhas de integridade
---

## Licença

Este projeto é para fins educacionais.