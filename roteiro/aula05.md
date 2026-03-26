## Aula 05 — Consultando Dados: `select()`, Filtros e Paginação

**Duração estimada:** 14 min

---

**[0:00 – 1:30] Gancho: o time de analytics quer os dados**

Nesta aula vamos aprender:

- Usar o `select()` moderno do SQLAlchemy 2.x
- Escolher o formato de retorno certo para cada situação
- Filtrar com `where()`, `and_()`, `or_()` e `like()`
- Ordenar e paginar resultados
- Usar IA para transformar perguntas em queries

O time de vendas pediu relatórios: 
* os 5 produtos mais caros de uma categoria a de Eletrônicos;
* quais clientes são de de SP ou RJ;
* pedidos paginados em 20 por página. 

Vamos carregar as bibliotecas e criar a conexão com o nosso banco de dados.

E vamos inspecionar nossas tabelas:

```python
# verificar tabelas existentes
insp = inspect(engine)
print("Tabelas existentes:", insp.get_table_names())
```

---

**[1:30 – 4:30] O básico do `select()` e os três formatos de retorno**

No SQLAlchemy 2.x, toda consulta começa com `select()`.  

A função `select()` apenas **monta** a query em memória, ela só vai ao banco quando você chama `execute()` ou `scalars()`.

Vamos ver três formatos de selecionar dados:

O primeiro é o `scalars()` → usamos quando queremos trabalhar com os **objetos completos** acessar atributos, navegar por relacionamentos:

```python
with Session(engine) as session:  # Abre uma sessão para interagir com o banco de dados
    clientes = session.scalars(select(Cliente)).all()  # Seleciona todos os registros da tabela Cliente como objetos ORM e os carrega na memória

    print(f"Total de clientes: {len(clientes)}")  # Imprime o número total de clientes encontrados
    for c in clientes:  # Itera objetos Cliente para exibir detalhes de cada um
        print(f"  {c.id} | {c.nome} | {c.cidade}/{c.estado}")  # Exibe o ID, nome e cidade/estado de cada cliente formatado
```

Usamos o `execute().all()` → quando só precisamos de algumas colunas.

```python
with Session(engine) as session:
    # Seleciona apenas as colunas necessárias
    stmt = select(Cliente.id, Cliente.nome, Cliente.estado)
    linhas = session.execute(stmt).all()

    print("ID  | Nome                 | Estado")
    print("-" * 40)
    for linha in linhas:
        print(f"{linha.id:<4}| {linha.nome:<21}| {linha.estado}")
```

Usamos o `execute().mappings().all()` quando queremos acessar campos pelos nomes, ótimo para passar para a biblioteca Pandas.

```python
with Session(engine) as session:  # Abre uma sessão para executar a query no banco
    stmt = select(Cliente.id, Cliente.nome, Cliente.cidade, Cliente.estado)  # Monta a query para selecionar colunas específicas da tabela Cliente
    dicts = session.execute(stmt).mappings().all()  # Executa a query e retorna os resultados como uma lista de dicionários (mappings)

    # Cada item é como um dicionário
    if dicts:  # Verifica se há resultados (lista não vazia)
        print("Primeiro registro como dicionário:")  # Imprime mensagem se houver dados
        print(dict(dicts[0]))  # Converte o primeiro dicionário para dict (redundante, mas mostra o formato) e imprime
```
---

**[4:30 – 8:30] Filtrando resultados**

Vamos agora aprender como filtrar dados usando a função `where`.

```python
with Session(engine) as session:

    # Clientes de SP
    stmt = select(Cliente).where(Cliente.estado == "SP")
    clientes_sp = session.scalars(stmt).all()
    print(f"Clientes em SP: {len(clientes_sp)}")

    # Encadeando múltiplos where() — funciona como AND
    stmt2 = (
        select(Cliente)
        .where(Cliente.estado == "SP")
        .where(Cliente.cidade == "São Paulo")
    )
    clientes_sao_paulo = session.scalars(stmt2).all()
    print(f"Clientes na cidade de São Paulo: {len(clientes_sao_paulo)}")
```

Agora, vamos aprender filtrar usando a condição `OR`.

```python
with Session(engine) as session:

    # Clientes de SP OU RJ
    stmt = select(Cliente).where(
        or_(Cliente.estado == "SP", Cliente.estado == "RJ")
    )
    # Alternativa mais pythônica para listas:
    # .where(Cliente.estado.in_(["SP", "RJ"]))

    clientes = session.scalars(stmt).all()
    print(f"Clientes em SP ou RJ: {len(clientes)}")
```

Agora, vamos aprender filtrar usando a função `LIKE`. LIKE conseguimos selecionar buscando por parte de uma palavra.

```python
with Session(engine) as session:

    # Produtos cujo nome contém "Note"
    stmt = select(Produto).where(Produto.nome.like("%Note%"))
    produtos = session.scalars(stmt).all()

    for p in produtos:
        print(f"  {p.nome} — R${p.preco_atual}")

    if not produtos:
        print("Nenhum produto encontrado (popule o banco primeiro)")
```

Agora, vamos montar queries dinamicamente

Pois no nosso dia a dia detrabalho, muitas vezes os filtros são opcionais e (vêm de um formulário, de uma API...).  

Por isso, vamos construir uma função para ser utilizada no nosso dia a dia em produção.

```python
def buscar_produtos(session, categoria=None, preco_max=None, apenas_ativos=True):
    """Busca produtos com filtros opcionais — padrão comum em APIs."""
    
    stmt = select(Produto)  # começa sem filtros

    if apenas_ativos:
        stmt = stmt.where(Produto.ativo == True)

    if categoria:
        stmt = stmt.where(Produto.categoria == categoria)

    if preco_max is not None:
        stmt = stmt.where(Produto.preco_atual <= preco_max)

    return session.scalars(stmt).all()


with Session(engine) as session:
    # Diferentes combinações de filtros
    todos_ativos = buscar_produtos(session)
    eletronicos  = buscar_produtos(session, categoria="Eletrônicos")
    baratos      = buscar_produtos(session, preco_max=Decimal("1000"))

    print(f"Todos os ativos:     {len(todos_ativos)}")
    print(f"Eletrônicos:         {len(eletronicos)}")
    print(f"Até R$1.000,00:      {len(baratos)}")
```
---

**[8:30 – 11:00] Ordenando e limitando resultados**

Agora vamos aprender como ordenar resultados:

```python
with Session(engine) as session:  # Abre uma sessão para executar a query no banco

    # Produtos ordenados por preço (maior primeiro)
    stmt = (  # Monta a query usando select() com encadeamento de métodos
        select(Produto.nome, Produto.categoria, Produto.preco_atual)  # Seleciona nome, categoria e preço dos produtos
        .where(Produto.ativo == True)  # Filtra apenas produtos ativos (ativo = True)
        .order_by(Produto.preco_atual.desc())  # Ordena por preço decrescente (maior primeiro); use .asc() para crescente
    )

    produtos = session.execute(stmt).all()  # Executa a query e retorna todos os resultados como tuplas

    print(f"{'Produto':<30} {'Categoria':<20} {'Preço':>10}")  # Imprime o cabeçalho da tabela formatado
    print("-" * 62)  # Imprime uma linha separadora
    for p in produtos[:5]:  # Itera apenas nos primeiros 5 produtos para limitar o output
        print(f"{p.nome:<30} {p.categoria:<20} R${p.preco_atual:>8}")  # Imprime cada produto formatado (nome alinhado à esquerda, categoria, preço à direita)
```
---

**[11:00 – 14:00] Paginação com `limit()` e `offset()`**

Agora vamos ver como podemos visualizar dados carregando aos poucos eles:

Temos a função `limit` que define quantos dados por página;
e o `offset` que define a partir de qual linha. 

Vamos ver na prática como funciona:

```python
def listar_clientes_paginados(session, pagina: int = 20, por_pagina: int = 5):  # Define função para listar clientes paginados
    """Retorna clientes paginados, ordenados por nome.""" 
    
    offset = (pagina - 1) * por_pagina  # Calcula o offset baseado na página e itens por página

    stmt = (  # Monta a query SQLAlchemy
        select(Cliente)  # Seleciona a tabela Cliente
        .order_by(Cliente.nome)  # Ordena os resultados por nome
        .limit(por_pagina)  # Limita o número de resultados por página
        .offset(offset)  # Pula os registros anteriores à página atual
    )

    return session.scalars(stmt).all()  # Executa a query e retorna lista de objetos Cliente


with Session(engine) as session:  # Abre uma sessão para executar queries
    # Total de clientes para calcular o total de páginas
    total = session.scalar(select(func.count()).select_from(Cliente))  # Conta o total de clientes na tabela
    print(f"Total de clientes: {total}")  # Imprime o total de clientes

    pagina_1 = listar_clientes_paginados(session, pagina=1, por_pagina=5)  # Chama a função para a página 1
    print(f"\nPágina 1 (5 por página):")  # Imprime cabeçalho da página
    for c in pagina_1:  # Itera sobre os clientes da página
        print(f"  {c.nome} — {c.cidade}/{c.estado}")  # Imprime nome e localização de cada cliente
```
---

**Exercício com IA**

Agora vamos usar a IA para mma das melhores aplicações dela no nosso dia a dia de dados: 

Que é **transformar perguntas em código de query**.

**Prompt para traduzir perguntas em queries:**
```
Usando os modelos SQLAlchemy abaixo, escreva uma query Python que responda:
"Quais são os 5 produtos mais vendidos do mês passado,
agrupados por categoria, com o total de receita de cada?"

[cole os modelos ORM aqui]
```
---
