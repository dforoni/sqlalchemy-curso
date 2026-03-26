## Aula 07 — Consultas Compostas: UNION, Subconsultas, IN e EXISTS

**Duração estimada:** 15 min

---

**[0:00 – 1:30] Gancho: perguntas que uma query simples não responde**

Nesta aula vamos aprender:

- `UNION` / `UNION ALL` — combinar conjuntos de resultados
- `subquery()` — usar uma query como etapa intermediária
- `IN` / `NOT IN` — filtrar por pertencimento
- `EXISTS` / `NOT EXISTS` — filtrar por existência

O gerente comercial pediu para nós: 

* os pedidos recentes OU de alto valor;
* clientes que compraram Eletrônicos;
* produtos que nunca foram vendidos;
* Quais categorias têm mais de 3 produtos cadastrados.

Primeiro de tudo, vamos importar e criar a conexão com o nosso banco de dados.

---

**[1:30 – 3:30] Construindo queries em etapas**

Agora, antes de entrarmos nas consultas compostas, um padrão útil: é o `select()` pode ser **construído aos poucos**.  

Como ele só vai ao banco quando você chama `execute()`, você pode ir adicionando filtros conforme a lógica exige:

```python
# A query é construída em etapas — vai ao banco apenas no execute()
stmt = select(Pedido.id, Pedido.status, Pedido.valor_total)

# Adicionando filtros conforme a lógica
stmt = stmt.where(Pedido.status != "Cancelado")
stmt = stmt.order_by(Pedido.valor_total.desc())
stmt = stmt.limit(10)

# Visualizando o SQL antes de executar — ótimo para debug!
print("SQL gerado:")
print(stmt)

with Session(engine) as session:
    resultados = session.execute(stmt).all()
    print(f"\nResultados: {len(resultados)} linhas")
```

---

**[3:30 – 9:30] UNION e UNION ALL**

Vamos agora combinar dois conjuntos e para isso usaremos o `union`, ele junta resultados de 2 queries em uma lista única.

Uma **regra:** as duas queries precisam retornar as **mesmas colunas** (mesmos tipos, mesma ordem).

Vamos ver um exemplo:

```python
# Calcula a data de 7 dias atrás subtraindo 7 dias da data/hora atual; usada como ponto de corte para filtrar pedidos recentes
data_corte = datetime.now() - timedelta(days=7)

# Query 1: pedidos recentes
data_corte = datetime.now() - timedelta(days=7)  
recentes = select(
    Pedido.id,
    Pedido.status,
    Pedido.valor_total,
).where(Pedido.data_pedido >= data_corte)

# Query 2: pedidos de alto valor
alto_valor = select(
    Pedido.id,
    Pedido.status,
    Pedido.valor_total,
).where(Pedido.valor_total >= 2000)

# UNION remove duplicatas — um pedido recente E de alto valor aparece uma vez só
stmt_union = union(recentes, alto_valor).order_by("valor_total")

with Session(engine) as session:
    prioritarios = session.execute(stmt_union).all()
    print(f"Pedidos prioritários (union): {len(prioritarios)}")

    # UNION ALL mantém duplicatas — útil para contar ocorrências por critério
    stmt_union_all = union_all(recentes, alto_valor)
    todos = session.execute(stmt_union_all).all()
    print(f"Com union_all (com duplicatas): {len(todos)}")
```

Agora vamos usar a subquery:

```python
# Passo 1: montar a query intermediária (IDs dos pedidos prioritários)
ids_prioritarios = union(recentes, alto_valor).subquery()

# Passo 2: usar a subquery para buscar os objetos Pedido completos
stmt = (
    select(Pedido)
    .join(ids_prioritarios, Pedido.id == ids_prioritarios.c.id)
)

with Session(engine) as session:
    pedidos_completos = session.scalars(stmt).all()
    print(f"Objetos Pedido completos: {len(pedidos_completos)}")
    for p in pedidos_completos[:3]:
        print(f"  #{p.id} | {p.status} | R${p.valor_total}")
```

---

**[9:30 – 12:30] IN, NOT IN, EXISTS e NOT EXISTS**

Agora, vamos aprender a filtrar por pertencimento:

Vamos usar o `IN`: o in percorre uma lista e retorna o que der match

```python
# Quais clientes compraram produtos de Eletrônicos?
# Caminho: Produto → ItemPedido → Pedido → Cliente

# Passo 1: IDs de produtos de Eletrônicos
ids_eletronicos = select(Produto.id).where(Produto.categoria == "Eletrônicos")

# Passo 2: IDs de pedidos que contêm esses produtos
ids_pedidos = select(ItemPedido.pedido_id).where(
    ItemPedido.produto_id.in_(ids_eletronicos)
)

# Passo 3: clientes desses pedidos
stmt = (
    select(Cliente)
    .where(Cliente.id.in_(
        select(Pedido.cliente_id).where(Pedido.id.in_(ids_pedidos))
    ))
)

with Session(engine) as session:
    clientes = session.scalars(stmt).all()
    print(f"Clientes que compraram Eletrônicos: {len(clientes)}")
    for c in clientes[:5]:
        print(f"  {c.nome} — {c.cidade}/{c.estado}")

# IN com lista fixa também funciona:
# .where(Pedido.status.in_(["Pago", "Enviado"]))
```
Vamos ver o uso do `EXISTS`, ele verifica a existência — por exemplo: "há ao menos um registro relacionado?" 

Vamos ver um exemplo: Clientes que fizeram ao menos um pedido de alto valor ("high ticket")

```python
# Clientes que fizeram ao menos um pedido de alto valor ("high ticket")
subq_alto_valor = (
    select(Pedido.id)
    .where(
        and_(
            Pedido.cliente_id == Cliente.id,  # correlação com a query externa
            Pedido.valor_total >= 1000
        )
    )
)

stmt = select(Cliente).where(exists(subq_alto_valor))

with Session(engine) as session:
    clientes_vip = session.scalars(stmt).all()
    print(f"Clientes com pedido >= R$1.000: {len(clientes_vip)}")


# NOT EXISTS — produtos que NUNCA foram vendidos
subq_vendido = (
    select(ItemPedido.produto_id)
    .where(ItemPedido.produto_id == Produto.id)
)

stmt_nao_vendidos = select(Produto).where(~exists(subq_vendido))

with Session(engine) as session:
    nao_vendidos = session.scalars(stmt_nao_vendidos).all()
    print(f"\nProdutos nunca vendidos: {len(nao_vendidos)}")
    for p in nao_vendidos[:3]:
        print(f"  {p.nome} — {p.categoria}")
```
---

**[12:30 – 15:00] **Exercício com IA**

Consultas compostas são um ótimo caso de uso para IA especialmente para decompor perguntas complexas.

**Prompt para decompor uma pergunta em query:**
```
Com os modelos SQLAlchemy abaixo, escreva a query para responder:
"Quais clientes que compraram na última semana NÃO compraram nenhum
produto da categoria 'Promoção'?"
Use subconsultas com IN ou NOT EXISTS conforme apropriado.
Explique cada etapa.
---