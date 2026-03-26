## Aula 06 — Relacionamentos, N+1 e Entregando Dados ao Pandas

**Duração estimada:** 14 min

---

**[0:00 – 1:30] Gancho: o dashboard está lento**

Nesta aula vamos aprender:

- Entender lazy vs eager loading na prática
- Identificar e corrigir o problema N+1
- Usar `joinedload` e `selectinload`
- Entregar resultados em DataFrames do Pandas

Estamos com um problema o dashboard de vendas demora 8 segundos. Investigamos e encontramos um loop disparando centenas de queries desnecessárias. 

Isso é o problema N+1 e vamos resolvê-lo agora.

Mas primeiro, vamos importar e criar a conexão com o nosso banco de dados.

---

**[1:30 – 4:30] Lazy loading e o problema N+1**

Por padrão, o SQLAlchemy usa **lazy loading**: os dados relacionados só são buscados quando você os acessa.  

Isso é conveniente para acessos pontuais, mas **desastroso em loops**.

Vamos ver um exemplo:

```python
# Para ver as queries sendo disparadas, ative o echo temporariamente
engine_verbose = create_engine("sqlite:///../database/datavendas.db", echo=True)

print("=" * 60)
print("LAZY LOADING: observe quantas queries são disparadas:")
print("=" * 60)

with Session(engine_verbose) as session:
    pedidos = session.scalars(select(Pedido)).all()
    # Até aqui: 1 query

    for p in pedidos[:3]:  # Limitando a 3 para não poluir o output
        # AQUI: uma nova query para cada pedido!
        nome_cliente = p.cliente.nome
        print(f"  Pedido #{p.id} → {nome_cliente}")
```

### O problema N+1 em números

Se você tem 200 pedidos e acessa `p.cliente` em loop:

```
1 query para buscar os 200 pedidos
+ 200 queries para buscar o cliente de cada pedido
= 201 queries  ← isso é o N+1!
```

Com banco local, cada query leva ~1ms. Com PostgreSQL em produção, pode levar 10-50ms.  
**200 × 50ms = 10 segundos**  é exatamente isso que estava travando o dashboard.

Execute o loop com `p.cliente` e mostre no log do `echo=True` a enxurrada de queries. Reforce: com PostgreSQL em produção, a 50ms por query, 200 queries = 10 segundos.

---

**[4:30 – 8:30] Eager loading: `joinedload` e `selectinload`**


A solução é avisar ao SQLAlchemy **antes de executar** que você vai precisar dos relacionamentos.  

O método `joinedload`: utiliza 1 query com JOIN. Usamos para relações muitos-para-um (pedido → cliente).

Exemplo:

```python
print("=" * 60)
print("EAGER LOADING com joinedload — observe: 1 única query:")
print("=" * 60)

with Session(engine_verbose) as session:
    stmt = (
        select(Pedido)
        .options(joinedload(Pedido.cliente))  # avisa: traga o cliente junto!
    )
    pedidos = session.scalars(stmt).all()

    for p in pedidos[:3]:
        # Sem queries adicionais — os dados já estão carregados!
        print(f"  Pedido #{p.id} → {p.cliente.nome}")
```


Já o métodos `selectinload`: utiliza 1 query principal + 1 `WHERE IN`. Usamos para coleções (cliente → lista de pedidos).

Vamos ver um exemplo:

```python
print("=" * 60)
print("selectinload — 2 queries para N pedidos com itens:")
print("=" * 60)

with Session(engine_verbose) as session:
    stmt = (
        select(Pedido)
        .options(
            joinedload(Pedido.cliente),        # cliente: join
            selectinload(Pedido.itens)         # itens: selectin (coleção)
            .joinedload(ItemPedido.produto)    # produto de cada item: join
        )
    )
    pedidos = session.scalars(stmt).unique().all()

    for p in pedidos[:2]:
        print(f"\nPedido #{p.id} | {p.cliente.nome} | R${p.valor_total}")
        for item in p.itens:
            print(f"  - {item.quantidade}x {item.produto.nome} @ R${item.preco_venda}")
```

---

**[8:30 – 12:00] Entregando resultados para o Pandas**

No nosso dia a dia como pessoas que trabalham com dados, a maioria das análises termina em um DataFrame. 

O fluxo ideal é: **fazer o recorte no banco** (eficiente) e trazer para Python apenas o necessário.

**Padrão recomendado:**
1. `select()` com as colunas necessárias
2. `mappings()` para obter dicionários dos dados
3. `pd.DataFrame()` para montar o DataFrame

```python
# Relatório de vendas por cliente
import pandas as pd  # Importa a biblioteca Pandas para trabalhar com DataFrames
with Session(engine) as session:  # Abre uma sessão para executar a query no banco
    stmt = (  # Monta a query SQLAlchemy para selecionar dados de vendas
        select(  # Seleciona colunas específicas de Cliente e Pedido
            Cliente.nome.label("cliente"),  # Nome do cliente, renomeado para "cliente"
            Cliente.estado,  # Estado do cliente
            Pedido.id.label("pedido_id"),  # ID do pedido, renomeado para "pedido_id"
            Pedido.data_pedido,  # Data do pedido
            Pedido.valor_total,  # Valor total do pedido
            Pedido.status,  # Status do pedido
        )
        .join(Pedido.cliente)  # Faz JOIN entre Pedido e Cliente via relacionamento ORM
        .order_by(Pedido.data_pedido.desc())  # Ordena os resultados por data do pedido decrescente (mais recentes primeiro)
    )

    dados = session.execute(stmt).mappings().all()  # Executa a query e retorna os resultados como uma lista de dicionários

df = pd.DataFrame(dados)  # Converte a lista de dicionários em um DataFrame do Pandas

print("DataFrame de vendas:")  # Imprime cabeçalho
print(df.head())  # Exibe as primeiras 5 linhas do DataFrame
print(f"\nShape: {df.shape}")  # Imprime o formato do DataFrame (linhas, colunas)
print(f"Colunas: {list(df.columns)}")  # Imprime a lista de colunas do DataFrame
```

```python
# Análise rápida após ter o DataFrame
if not df.empty:
    print("Vendas por status:")
    print(df.groupby("status")["valor_total"].agg(["count", "sum"]).round(2))
    
    print("\nTop 5 clientes por valor total:")
    top_clientes = (
        df.groupby("cliente")["valor_total"]
        .sum()
        .sort_values(ascending=False)
        .head(5)
    )
    print(top_clientes)
```
---

**Exercício com IA**

Vamos usar a IA para criar uma análise usando o SQLAlchemy e o Pandas.

**Prompt para gerar pipeline Pandas:**
```
Crie um pipeline SQLAlchemy + Pandas que:
1. Busque os pedidos do último mês com cliente e itens (sem N+1)
2. Converta para DataFrame
3. Calcule receita total por categoria de produto
4. Retorne o top 5 categorias
---