## Aula 09 — JOINs Avançados: INNER, LEFT, Múltiplos e Self-Join

**Duração estimada:** 15 min

---

**[0:00 – 1:30] Gancho: o relatório está errado**

Nesta aula vamos aprender:

- A diferença prática entre INNER JOIN e LEFT OUTER JOIN
- JOINs múltiplos encadeados
- Self-join: comparar a tabela com ela mesma
- JOIN + GROUP BY para métricas
- Como inspecionar o SQL gerado para diagnosticar erros

Estamos com um probelma o relatório mostra 120 clientes, mas o gerente sabe que são 150. Você investiga: o código usa `JOIN` onde deveria usar `LEFT JOIN`, excluindo clientes sem pedidos. 
Este é um bug silencioso clássico e vamos aprender a evitá-lo.

Agora, vamos importar e criar a conexão com o nosso banco de dados.

---

**[1:30 – 3:00] JOIN via relacionamento vs JOIN explícito**

O SQLAlchemy oferece duas formas de escreve um `JOIn`

- Via relacionamento: `join(Cliente.pedidos)` — use quando o relacionamento ORM existe e o JOIN é direto.

- Explícito: `join(Pedido, Cliente.id == Pedido.cliente_id)` — use quando precisar de condições extras no `ON`.

---

**[3:00 – 6:00] INNER JOIN vs LEFT OUTER JOIN**

O `INNER JOIN` funciona como a interseção de dois conjuntos. Ele percorre a tabela de `Clientes` e a tabela de `Pedidos` e só traz os resultados onde existe uma correspondência exata entre as duas, baseada na chave estrangeira.

Vamos ver um exemplo:

```python
with Session(engine) as session:

    # INNER JOIN padrão: retorna apenas clientes COM pedidos
    stmt = (
        select(
            Cliente.nome,
            Cliente.estado,
            Pedido.id.label("pedido_id"),
            Pedido.valor_total,
            Pedido.status,
        )
        .join(Cliente.pedidos)  # INNER JOIN via relacionamento
        .order_by(Pedido.valor_total.desc())
    )

    linhas = session.execute(stmt).all()
    total_clientes = session.scalar(select(func.count()).select_from(Cliente))

    print(f"Total de clientes no banco:    {total_clientes}")
    print(f"Linhas no INNER JOIN:          {len(linhas)}")
    print(f"Clientes sem pedido não aparecem!")   
    print("-" * 70)
    # Imprimindo as linhas
    for l in linhas:
        print(f"  {l.nome} | {l.estado} | Pedido #{l.pedido_id} | R${l.valor_total}")
```

### LEFT OUTER JOIN — todos da esquerda, com ou sem par

O `.outerjoin()` retorna **todos os clientes**, mesmo os sem pedido.  
Onde não há par, as colunas do lado direito aparecem como `None`.

Abaixo vamos ver um exemplo:

```python
with Session(engine) as session:  # Inicia uma sessão de banco de dados para executar consultas

    stmt = (  
        select(  # Seleciona colunas específicas das tabelas
            Cliente.nome,  # Nome do cliente (da tabela Cliente)
            Pedido.id.label("pedido_id"),  # ID do pedido, rotulado como "pedido_id" (da tabela Pedido)
            Pedido.valor_total,  # Valor total do pedido (da tabela Pedido)
        )
        .outerjoin(Cliente.pedidos)  # Realiza um LEFT OUTER JOIN entre Cliente e Pedido via relacionamento 'pedidos'
        .order_by(Cliente.nome)  # Ordena os resultados pelo nome do cliente em ordem alfabética
    )

    linhas = session.execute(stmt).all()  # Executa a consulta e obtém todas as linhas resultantes como uma lista de objetos
    print(f"Linhas no LEFT JOIN: {len(linhas)} (inclui clientes sem pedido)")  # Imprime o número total de linhas, incluindo clientes sem pedidos

    # Clientes sem pedido aparecem com None  
    sem_pedido = [l for l in linhas if l.pedido_id is None]  # Filtra a lista para obter apenas linhas onde pedido_id é None (clientes sem pedidos)
    print(f"Clientes sem nenhum pedido: {len(sem_pedido)}")  # Imprime o número de clientes que não têm nenhum pedido
```

---

**[6:00 – 9:00] JOINs múltiplos encadeados**

### JOINs múltiplos — encadeando tabelas

Em análises de vendas, a cadeia completa é:  
**Cliente → Pedido → ItemPedido → Produto**

Muitas vezes precisamos criar joins com múltiplas tabelas para conseguir construir a tabela que precisamos para as nossas análises. 

Abaixo vamos ver um exemplo:

```python
# Abre uma sessão de conexão com o banco de dados usando o engine configurado
with Session(engine) as session:

    stmt = (
        select(
            
            Cliente.nome.label("cliente"),          
            Pedido.id.label("pedido"),
            Produto.nome.label("produto"),
            Produto.categoria,
            ItemPedido.quantidade,
            ItemPedido.preco_venda,            
            # Calcula o subtotal (quantidade * preço) e renomeia como "subtotal"
            (ItemPedido.quantidade * ItemPedido.preco_venda).label("subtotal")
        )
        
        # Faz JOIN entre Cliente e Pedido (um cliente pode ter vários pedidos)
        .join(Pedido, Cliente.id == Pedido.cliente_id)        
        # Faz JOIN entre Pedido e ItemPedido (um pedido pode ter vários itens)
        .join(ItemPedido, Pedido.id == ItemPedido.pedido_id)        
        # Faz JOIN entre ItemPedido e Produto (cada item corresponde a um produto)
        .join(Produto, ItemPedido.produto_id == Produto.id)        
        # Ordena o resultado pelo valor do subtotal
        .order_by("subtotal")
    )

    # Executa a consulta e retorna todos os resultados
    linhas = session.execute(stmt).all()
    
    # Exibe a quantidade total de linhas retornadas
    print(f"Itens vendidos (com contexto completo): {len(linhas)}")
    print("-" * 80)

    # Percorre apenas os 3 primeiros resultados
    for l in linhas[:3]:
        
        # Exibe as informações formatadas de cada item
        print(f"  {l.cliente} → Pedido #{l.pedido} | {l.quantidade}x {l.produto} | R${l.subtotal}")
```
---

**[9:00 – 11:30] Self-join com `aliased()`**

No nosso dia a dia acabamos enfrentando algumas análises mais complexas: do tipo "quais pares de clientes moram na mesma cidade?" Para isso, precisamos comparar a tabela de clientes com ela mesma.

Para essa demanda usamos o método `aliased()`, ele cria duas cópias da mesma tabela com nomes diferentes. 

Vamos ver um exemplo: "quais pares de clientes moram na mesma cidade?

```python
# Cria dois aliases (apelidos) para a tabela Cliente
# Isso permite usar a mesma tabela duas vezes na mesma consulta (self join)
ClienteA = aliased(Cliente, name="a")
ClienteB = aliased(Cliente, name="b")

# Monta a consulta
stmt = (
    select(
        # Seleciona o nome do primeiro cliente e renomeia como "cliente_1"
        ClienteA.nome.label("cliente_1"),
        
        # Seleciona o nome do segundo cliente e renomeia como "cliente_2"
        ClienteB.nome.label("cliente_2"),
        ClienteA.cidade,
    )
    
    # Faz um JOIN da tabela Cliente com ela mesma (self join)
    # A condição é clientes que estão na mesma cidade
    .join(ClienteB, ClienteA.cidade == ClienteB.cidade)
    
    # Evita duplicatas como (A,B) e (B,A)
    # Também evita comparar o cliente com ele mesmo
    .where(ClienteA.id < ClienteB.id)
    
    # Ordena o resultado pela cidade
    .order_by(ClienteA.cidade)
)

# Abre uma sessão com o banco de dados
with Session(engine) as session:
    
    # Executa a consulta e retorna todos os resultados
    pares = session.execute(stmt).all()
    
    # Exibe a quantidade de pares encontrados
    print(f"Pares de clientes na mesma cidade: {len(pares)}")
    print("-" * 80)
    
    # Mostra apenas os 5 primeiros pares
    for p in pares[:5]:
        
        # Exibe os clientes que estão na mesma cidade
        print(f"  {p.cliente_1} + {p.cliente_2} — {p.cidade}")
```

---

**[11:30 – 13:30] JOIN + GROUP BY para métricas**

Agora veremos um exemplo de receita total por cliente usando JOIN + GROUP BY. 

Veremos no exemplo abaixo: qual a receita total por cliente (apenas quem tem pedidos)

```python
# Receita total por cliente (apenas quem tem pedidos)

# Abre uma sessão com o banco de dados
with Session(engine) as session:

    # Monta a consulta SQL
    stmt = (
        select(
         
            Cliente.nome,         
            Cliente.estado,            
            # Conta quantos pedidos cada cliente fez
            func.count(Pedido.id).label("qtd_pedidos"),            
            # Soma o valor total dos pedidos (receita total por cliente)
            func.sum(Pedido.valor_total).label("receita_total"),            
            # Calcula o ticket médio (média do valor dos pedidos)
            func.avg(Pedido.valor_total).label("ticket_medio"),
        )
        
        # Faz JOIN entre Cliente e Pedido
        # Apenas clientes com pedidos aparecerão no resultado
        .join(Pedido, Cliente.id == Pedido.cliente_id)        
        # Agrupa os resultados por cliente
        # Necessário para usar funções de agregação (count, sum, avg)
        .group_by(Cliente.id)        
        # Ordena pela maior receita total (ordem decrescente)
        .order_by(func.sum(Pedido.valor_total).desc())
    )

    # Executa a consulta e retorna todos os resultados
    resultados = session.execute(stmt).all()

    # Imprime o cabeçalho da tabela formatada
    print(f"{'Cliente':<25} {'UF'} {'Pedidos':>8} {'Receita':>12} {'Ticket Médio':>13}")
    
    # Imprime uma linha separadora
    print("-" * 65)
    
    # Percorre apenas os 8 primeiros resultados
    for r in resultados[:8]:
        
        # Exibe os dados formatados
        print(
            f"{r.nome:<25} "
            f"{r.estado}  "
            f"{r.qtd_pedidos:>8}  "
            f"R${float(r.receita_total or 0):>9.2f}  "
            f"R${float(r.ticket_medio or 0):>9.2f}"
        )
```
---

**[13:30 – 15:00] Visualizando o SQL gerado**

Agora veremos como visualizar o SQL gerado pelo ORM, isso é uma das melhores formas de aprender SQLAlchemy é ver o SQL real que ele está enviando para o banco de dados.

```python
from sqlalchemy.dialects import sqlite

# Compilando o comando especificamente para o dialeto SQLite
debug_sql = stmt.compile(dialect=sqlite.dialect(), compile_kwargs={"literal_binds": True})

print("--- SQL COM VALORES REAIS ---")
print(debug_sql)
```
---

**Exercício com IA**

Vamos usar a IA para ajudar a gente com um problema de lentidão nos nossos `JOINS`.

**Prompt para otimizar JOINs múltiplos:**
```
Tenho esta query SQLAlchemy com 3 JOINs encadeados que está lenta.
Sugira: índices que deveriam existir, reescrita mais eficiente,
ou se subconsulta seria mais adequada em alguma parte.
```