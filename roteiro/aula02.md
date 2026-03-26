## Aula 02 — Modelando o Banco com SQLAlchemy Core

**Duração estimada:** 14 min

---

**[0:00 – 1:30] Gancho: schema no controle de versão**

Nesta aula vamos ver: 

- Como: Criar tabelas com `MetaData` e `Table`
- Como: Definir colunas com tipos e restrições realistas
- Como: Usar `ForeignKey` para relacionar tabelas
- Como: Inspecionar o banco criado
- Como: Usar IA para gerar schemas a partir de descrições

Imagine que o nosso gerente passou o diagrama do banco e quer o schema criado via código Python. 

Por quê? Porque assim entra no conttole de versonamento do Git, fica documentado e é reproduzível em qualquer ambiente — dev, staging, produção.

Vamos importar as bibliotecas necessárias e aproveitar para criar a conexão com o nosso banco de dados SQLite.

---

**[1:30 – 3:30] MetaData — o catálogo do banco**

O método `MetaData` é como um catálogo central que registra todas as tabelas. 
Igual esse exemplo abaixo: com as tabelas clientes, pedidos, produtos e itens_pedidos
```
MetaData
  ├── tb_clientes
  ├── tb_pedidos
  ├── tb_produtos
  └── tb_itens_pedidos
```
---

**[3:30 – 8:00] Criando as quatro tabelas**

Agora vamos criar as 4 tabelas:

Vamos começar pela tabela **tb_clientes:** 

Cada `Column` define um campo com seu tipo e suas regras de integridade:

| Parâmetro | O que garante |
|---|---|
| `primary_key=True` | Identificador único da linha |
| `nullable=False` | Campo obrigatório — não pode ficar vazio |
| `unique=True` | Não pode ter valores repetidos na coluna |

```python
# Define a tabela de clientes com suas colunas e restrições
tb_clientes = Table(
    "tb_clientes",  # Nome da tabela no banco
    metadata,  # Registra no MetaData
    Column("id",            Integer,      primary_key=True),  # Chave primária, autoincremento
    Column("nome",          String(100),  nullable=False),    # Nome obrigatório, até 100 chars
    Column("email",         String(150),  nullable=False, unique=True),  # Email único e obrigatório
    Column("cidade",        String(80),   nullable=False),    # Cidade obrigatória
    Column("estado",        String(2),    nullable=False),    # Estado obrigatório (UF)
    Column("data_cadastro", DateTime,     nullable=False),    # Data de cadastro obrigatória
    extend_existing=True  # Permite redefinir se já existir no metadata
)

print("Tabela 'tb_clientes' definida ✅")
```

Agora vamos criar a **tb_produtos:** 

```python
# Define a tabela de produtos
tb_produtos = Table(
    "tb_produtos",
    metadata,
    Column("id",        Integer,       primary_key=True),  # ID único do produto
    Column("nome",      String(150),   nullable=False),    # Nome do produto
    Column("categoria", String(80),    nullable=False),    # Categoria do produto
    Column("preco_atual",     Numeric(10, 2), nullable=False),  # Preço com 2 casas decimais
    Column("ativo",     Boolean,        nullable=False, default=True),  # Se o produto está ativo
    extend_existing=True
)

print("Tabela 'tb_produtos' definida ✅")
```
> 💡 **Por que `Numeric` e não `Float`?**  
> `Float` é impreciso para dinheiro: `0.1 + 0.2 = 0.30000000000000004`.  
> `Numeric(10, 2)` garante exatidão decimal: use sempre para valores monetários.


Agora vamos criar a tabela **tb_pedidos:** 

```python
# Define a tabela de pedidos com foreign key e check constraint
tb_pedidos = Table(
    "tb_pedidos",
    metadata,
    Column("id",          Integer,                               primary_key=True),  # ID do pedido
    Column("cliente_id",  Integer, ForeignKey("tb_clientes.id"), nullable=False),  # FK para clientes
    Column("data_pedido", DateTime,                              nullable=False),   # Data do pedido
    Column("valor_total", Numeric(10, 2),                        nullable=False),   # Valor total
    Column("status",      String(20),                            nullable=False),   # Status do pedido

    # Constraint para validar valores de status
    CheckConstraint(
        "status IN ('Criado', 'Pago', 'Enviado', 'Cancelado')",
        name="ck_status_pedido"
    ),
    extend_existing=True
)

print("Tabela 'tb_pedidos' definida ✅")
```

Aqui aparecem dois conceitos importantes:
- **`ForeignKey`**: conecta um pedido a um cliente (relação N:1)
- **`CheckConstraint`**: garante que o `status` só aceite valores do domínio definido

Agora vamos criar a tabela **tb_itens_pedidos:** 

Esta tabela representa a relação **N:N** entre pedidos e produtos.  

É a tabela associativa clássica: um pedido tem vários produtos, e um produto pode estar em vários pedidos.

```python
# Define a tabela associativa para itens dos pedidos (relação N:N)
tb_itens_pedidos = Table(
    "tb_itens_pedidos",
    metadata,
    Column("id",          Integer,                               primary_key=True),  # ID do item
    Column("pedido_id",   Integer, ForeignKey("tb_pedidos.id"),  nullable=False),  # FK para pedidos
    Column("produto_id",  Integer, ForeignKey("tb_produtos.id"), nullable=False),  # FK para produtos
    Column("quantidade",  Integer,                               nullable=False),   # Quantidade comprada
    Column("preco_venda",  Numeric(10, 2),                        nullable=False),   # Preço na venda
    extend_existing=True
)

print("Tabela 'tb_itens_pedidos' definida ✅")
```

---

**[8:00 – 11:00] Criando as tabelas no banco com `create_all`**

Até agora, tudo existia apenas **em memória** (no objeto `metadata`).  

O `create_all` envia os comandos `CREATE TABLE` para o banco de verdade.

> `create_all` é seguro para rodar múltiplas vezes: ele só cria tabelas que **ainda não existem**.

```python
# Cria todas as tabelas definidas no MetaData no banco de dados
metadata.create_all(engine)
print("Todas as tabelas criadas no banco! ✅")
```

Agora vamos usar o `Inspect` para confirmar que as quatro tabelas foram criadas. 
Esse módulo é muito útil quando pegamos bancos legados, bancos que não fomos nós que criarmos.

```python
# Cria um inspetor para examinar a estrutura do banco
inspector = inspect(engine)

# Itera sobre todas as tabelas do banco
for tabela in inspector.get_table_names():
    print(f"\nTabela: {tabela}")
    print(f"   {'Coluna':<25} {'Tipo':<20} {'Nullable':<10}")
    print(f"   {'-'*55}")
    # Para cada tabela, lista suas colunas com detalhes
    for col in inspector.get_columns(tabela):
        print(f"   {col['name']:<25} {str(col['type']):<20} {str(col['nullable']):<10}")

    # Verifica se há foreign keys e as lista
    fks = inspector.get_foreign_keys(tabela)
    if fks:
        print(f"   🔗 Foreign Keys:")
        for fk in fks:
            print(f"      {fk['constrained_columns']} → {fk['referred_table']}.{fk['referred_columns']}")
```

---

**[11:00 – 14:00] Exercício com IA**

Agora vamos para o nosso exercício, vamos resolver ele novamente com a IA

O nosso prompt é para gerar um schema de RH (Departamento, Funcionário, Cargo) com IA. 

Utilizei para gerar esses códigos o Copilot, no nosso dia a dia como pessoas que trabalham com dados, o uso de IA facilita a nossa produtividade,mas sempre precisamos ficar atentos para verificar se a resposta da IA está correta.

---