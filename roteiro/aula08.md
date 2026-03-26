## Aula 08 — Filtrando Grupos: HAVING e ANY

**Duração estimada:** 13 min

---

**[0:00 – 1:30] Gancho: perguntas sobre totais**

Nesta aula vamos aprender:

- `HAVING` — filtrar resultados agrupados com condições sobre agregações
- `ANY` — comparar um valor com ao menos um elemento de uma subconsulta

Desta vez o gerente pediu: 

- *"Quais categorias têm mais de 3 produtos cadastrados?"*
- *"Quais clientes fizeram mais de 2 pedidos no total?"*
- *"Quais produtos têm preço maior que pelo menos um item vendido abaixo de R$50?"*

Vamos importar e criar a conexão com o nosso banco de dados.

---

**[1:30 – 4:00] WHERE vs HAVING — a distinção fundamental**

Use a analogia do notebook: `WHERE` descarta cédulas inválidas **antes** de contar. `HAVING` elimina candidatos com poucos votos **depois** de contar.

Mostre a tabela comparativa:

| | WHERE | HAVING |
|---|---|---|
| Filtra | Linhas individuais | Grupos (depois do GROUP BY) |
| Pode usar `func.count()`? | ❌ Não | ✅ Sim |

---

**[4:00 – 10:30] Casos de uso com HAVING**

O `HAVING` filtra grupos depois do `GROUP BY`, da mesma forma que o `WHERE` filtra linhas individuais, mas aplicado sobre o resultado de uma agregação.

**Por exemplo:** se você agrupar pedidos por cliente e contar quantos cada um fez, o `HAVING` permite manter só os grupos onde essa contagem é maior que 2. Isso não é possível com WHERE, porque a contagem ainda não existe quando o WHERE é aplicado.

Resumindo: `WHERE` filtra antes de agrupar, `HAVING` filtra depois.

Agora vamos responder a pergunta do gestor: Quais categorias têm mais de 3 produtos cadastrados?

```python
# Quais categorias têm mais de 3 produtos cadastrados?

total_produtos = func.count(Produto.id) # conta quantos produtos existem

stmt = (
    select(
        Produto.categoria,
        total_produtos.label("total_produtos")
    )
    .group_by(Produto.categoria) # agrupa por categoria
    .having(total_produtos > 3) # filtra os grupos
    .order_by(total_produtos.desc())
)

# Debug
print(stmt.compile(compile_kwargs={"literal_binds": True}))
print("-" * 70)
with Session(engine) as session:
    resultados = session.execute(stmt).all()

    for row in resultados:
        print(f"{row.categoria}: {row.total_produtos} produtos")
```

Agora, vamos combinar o `WHERE` e o `HAVING`na mesma query:

Os dois podem coexistir. `WHERE` filtra as linhas brutas; `HAVING` filtra os grupos resultantes.

Vamos responder a outra pergunta do gerente: clientes ativos (com pedido não cancelado) que fizeram mais de 2 pedidos.

```python
# WHERE filtra pedidos cancelados ANTES de agrupar.
# HAVING filtra os grupos com poucos pedidos DEPOIS de agrupar.

stmt = (
    select(
        Pedido.cliente_id,
        func.count(Pedido.id).label("total_pedidos"),
        func.sum(Pedido.valor_total).label("valor_acumulado")
    )
    .where(Pedido.status != "Cancelado")           # filtra LINHAS
    .group_by(Pedido.cliente_id)
    .having(func.count(Pedido.id) > 2)             # filtra GRUPOS
    .order_by(func.sum(Pedido.valor_total).desc())
)

print("SQL gerado:")
print(stmt)

with Session(engine) as session:
    grupos = session.execute(stmt).all()
    print(f"\nClientes com mais de 2 pedidos válidos: {len(grupos)}")
    for cliente_id, total, valor in grupos:
        print(f"  Cliente #{cliente_id} → {total} pedidos | R${valor:.2f} acumulado")
```

Agora vamos usar o `HAVING` para retornar apenas os campos do `GROUP BY`.

Exemplo buscar apenas clientes com pedidos não cancelados que tiveram mais que 2 pedidos.

```python
# Passo 1: subconsulta para obter apenas os IDs dos clientes qualificados
ids_clientes_freq = (
    select(Pedido.cliente_id)
    .where(Pedido.status != "Cancelado")
    .group_by(Pedido.cliente_id)
    .having(func.count(Pedido.id) > 2)
)

# Passo 2: buscar os objetos Cliente completos
stmt = select(Cliente).where(Cliente.id.in_(ids_clientes_freq))

with Session(engine) as session:
    clientes_freq = session.scalars(stmt).all()
    print(f"Clientes frequentes: {len(clientes_freq)}")
    for c in clientes_freq:
        print(f"  {c.nome} — {c.cidade}/{c.estado}")
```

---

**[10:30 – 13:00] ANY — comparar com ao menos um valor**

O `ANY` responde: *"este valor é maior/menor/igual a **pelo menos um** valor dessa subconsulta?"*

Funciona com qualquer operador de comparação: `>`, `<`, `=`, `!=`, `>=`, `<=`.

| Expressão | Significado |
|---|---|
| `col > any_(subq)` | `col` é maior que **ao menos um** valor da subconsulta |
| `col = any_(subq)` | `col` é igual a **ao menos um** valor (equivale ao `IN`) |
| `col < any_(subq)` | `col` é menor que **ao menos um** valor |


> ⚠️ **Nota SQLite:** `ANY` é um operador do padrão SQL e funciona nativamente em PostgreSQL e MySQL. No SQLite, o SQLAlchemy emula o comportamento por baixo para produção com SQLite, prefira `IN` ou `EXISTS`.


Vamos responder agora a outra pergunta do gestor: produtos com preço acima de algum item vendido a preço baixo

```python
# Subquery: valores dos pedidos cancelados
subq_cancelados = (
    select(Pedido.valor_total)
    .where(Pedido.status == "Cancelado")
)

# Query principal
stmt = (
    select(Cliente.nome, Pedido.valor_total)
    .join(Pedido)
    .where(Pedido.valor_total > any_(subq_cancelados))
)

print(stmt.compile(compile_kwargs={"literal_binds": True}))

with Session(engine) as session:
    resultados = session.execute(stmt).all()

    for nome, valor in resultados:
        print(f"{nome} fez um pedido de R$ {valor} acima de algum pedido cancelado")
```

A alternativa ao `ANY`é utilizar a função `MIN`.

```python
# alternativa ao ANY, vamos usar o MIN
# Subconsulta para obter o menor valor_total dos pedidos cancelados
subq = (
    select(func.min(Pedido.valor_total))  # Seleciona o valor mínimo de valor_total
    .where(Pedido.status == "Cancelado")  # Filtra apenas pedidos com status "Cancelado"
)

# Consulta principal: seleciona clientes cujos pedidos têm valor_total maior que o mínimo dos cancelados
stmt_min = (
    select(Cliente.nome, Pedido.valor_total)  # Seleciona o nome do cliente e o valor total do pedido
    .join(Pedido)  # Faz join com a tabela Pedido (assumindo relacionamento Cliente-Pedido)
    .where(Pedido.valor_total > subq.scalar_subquery())  # Filtra pedidos onde valor_total > mínimo dos cancelados
)

# Imprime o SQL gerado pela consulta (com valores literais para debug)
print(stmt_min.compile(compile_kwargs={"literal_binds": True}))
print("-" * 80)

# Executa a consulta no banco de dados
with Session(engine) as session:
    resultados = session.execute(stmt_min).all()  # Obtém todos os resultados como tuplas

    # Itera sobre os resultados e imprime cada cliente com seu pedido
    for nome, valor in resultados:
        print(f"{nome} fez um pedido de R$ {valor} acima de algum pedido cancelado")
```
---

**Exercício com IA**

Vamos utilizar a IA para ajudar a gente a responder outra pergunta do gerente de vendas:

**Prompt para gerar um HAVING:**
```
Com os modelos SQLAlchemy abaixo, escreva uma query que retorne
os estados com mais de 2 clientes cadastrados, ordenados
pelo total de clientes decrescente.
Use GROUP BY + HAVING.
```