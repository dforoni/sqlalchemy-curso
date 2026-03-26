## Aula 03 — ORM: Modelos e Sessão

**Duração estimada:** 15 min

---

**[0:00 – 1:30] Gancho: inserindo dados com objetos Python**

Nesta aula vamos aprender:

- Como: Criar modelos ORM com `DeclarativeBase`
- Como: Usar a `Session` para operações CRUD
- Como: Entender transações (commit/rollback)
- Como: Trabalhar com relacionamentos entre objetos
- Como: Usar IA para gerar modelos a partir de schemas

Agora, vamos imaginar que o nosso gerente passou os dados iniciais e pediu para serem inseridos no banco via código. 

O ORM permite trabalhar com classes e objetos em vez de strings SQL — mais fácil de manter e menos propenso a erros.

Então vamos iniciar carregando as bibliotecas necessárias

Mas o que importamos: os imports do ORM são diferentes dos do Core: os principais

| Import | Para que serve |
|---|---|
| `DeclarativeBase` | Classe mãe de todos os modelos ORM |
| `Mapped[tipo]` | Anotação de tipo — define coluna + tipo Python |
| `mapped_column(...)` | Configurações da coluna (nullable, unique, FK...) |
| `relationship(...)` | Navegação entre tabelas via objetos Python |
| `Session` | Gerencia as transações com o banco |

---

**[1:30 – 4:00] DeclarativeBase e os quatro modelos**

### Mas O que é a `DeclarativeBase`?

É o ponto de ancoragem de todos os modelos. Quando você cria uma classe que herda de `Base`, o SQLAlchemy sabe que ela representa uma tabela no banco.

```python
class Base(DeclarativeBase):
    pass

class Cliente(Base):   # ← SQLAlchemy reconhece como tabela
    ...
```

Todos os modelos do projeto herdam da **mesma** `Base` isso é o que permite ao `create_all()` criar todas as tabelas de uma vez e ao ORM montar os JOINs automaticamente.

```python
# DeclarativeBase é a classe-mãe que conecta todos os modelos ORM ao banco
# Todos os modelos do projeto herdarão desta mesma Base
class Base(DeclarativeBase):
    pass

print("DeclarativeBase criada ✅")
```

Agora vamos criar o modelo Cliente:

```python
# Modelo ORM para a tabela de clientes
class Cliente(Base):
    __tablename__ = "tb_clientes"

    # Mapped[tipo] define o tipo Python — o SQLAlchemy infere o tipo SQL
    id:            Mapped[int]      = mapped_column(primary_key=True)
    nome:          Mapped[str]      = mapped_column(String(100), nullable=False)
    email:         Mapped[str]      = mapped_column(String(150), nullable=False, unique=True)
    cidade:        Mapped[str]      = mapped_column(String(80),  nullable=False)
    estado:        Mapped[str]      = mapped_column(String(2),   nullable=False)
    data_cadastro: Mapped[datetime] = mapped_column(DateTime,    nullable=False)

    # Relacionamento: um cliente pode ter vários pedidos
    pedidos: Mapped[list["Pedido"]] = relationship(back_populates="cliente")

    def __repr__(self):
        return f"Cliente(id={self.id}, nome='{self.nome}', cidade='{self.cidade}')"

print("Modelo Cliente definido ✅")
```
Cada atributo da classe vira uma coluna no banco. O `Mapped[tipo]` faz dois trabalhos ao mesmo tempo: define o tipo Python **e** informa ao SQLAlchemy qual tipo SQL usar.

```
Mapped[int]      →  INTEGER
Mapped[str]      →  VARCHAR
Mapped[datetime] →  DATETIME
Mapped[Decimal]  →  NUMERIC
Mapped[bool]     →  BOOLEAN
```

O `relationship("Pedido", ...)` não cria uma coluna, ele cria um **atalho de navegação** em Python: `cliente.pedidos` retorna a lista de pedidos sem você escrever nenhum JOIN.

Agora vamos criar o modelo Produto:

```python
# Modelo ORM para a tabela de produtos
class Produto(Base):
    __tablename__ = "tb_produtos"

    # Mapped[tipo] define o tipo Python — o SQLAlchemy infere o tipo SQL
    id:        Mapped[int]     = mapped_column(primary_key=True)
    nome:      Mapped[str]     = mapped_column(String(150),    nullable=False)
    categoria: Mapped[str]     = mapped_column(String(80),     nullable=False)
    preco_atual:     Mapped[Decimal] = mapped_column(Numeric(10, 2), nullable=False)
    ativo:     Mapped[bool]    = mapped_column(Boolean,        nullable=False, default=True)

    # Relacionamento: um item pedido pode ter vários produtos
    itens: Mapped[list["ItemPedido"]] = relationship(back_populates="produto")

    def __repr__(self):
        # Essa parte serve apenas para mostrar o objeto de forma legível quando imprimimos
        return f"Produto(id={self.id}, nome='{self.nome}', preco={self.preco_atual})"

print("Modelo Produto definido ✅")
```
Repare no campo `ativo: Mapped[bool]` — no banco isso vira uma coluna `BOOLEAN`, mas no Python você trabalha com `True` / `False` de forma natural.

Agora vamos criar o modelo Pedido:

```python
# Modelo ORM para a tabela de pedidos
class Pedido(Base):
    __tablename__ = "tb_pedidos"
    # __table_args__ permite adicionar constraints ao nível da tabela
    # CheckConstraint garante que status só aceita valores válidos
    __table_args__ = (
        CheckConstraint("status IN ('Criado', 'Pago', 'Enviado', 'Cancelado')", name="ck_status_pedido"),
    )

    id:          Mapped[int]      = mapped_column(primary_key=True)
    cliente_id:  Mapped[int]      = mapped_column(ForeignKey("tb_clientes.id"), nullable=False)
    data_pedido: Mapped[datetime] = mapped_column(DateTime,       nullable=False)
    valor_total: Mapped[Decimal]  = mapped_column(Numeric(10, 2), nullable=False)
    status:      Mapped[str]      = mapped_column(String(20),     nullable=False)

    # Navegação bidirecional: pedido sabe quem é o cliente, cliente sabe seus pedidos
    cliente: Mapped[Cliente]            = relationship(back_populates="pedidos")
    itens:   Mapped[list["ItemPedido"]] = relationship(back_populates="pedido")

    def __repr__(self):
        return f"Pedido(id={self.id}, status='{self.status}', total={self.valor_total})"

print("Modelo Pedido definido ✅")
```

Dois detalhes importantes neste modelo:

**`__table_args__` com `CheckConstraint`:** permite adicionar restrições que não cabem nos parâmetros de `mapped_column`. Aqui garantimos que o `status` só aceite os 4 valores definidos.

**Dois `relationship()`:** `cliente` e `itens`. Isso cria a navegação nos dois sentidos:
```python
pedido.cliente.nome       # acessa o cliente do pedido
pedido.itens              # lista os itens do pedido
```
O `back_populates` mantém os dois lados sincronizados se você adicionar um pedido a um cliente, `cliente.pedidos` se atualiza automaticamente.


Agora vamos criar o modelo ItemPedido:

O `ItemPedido` é a tabela que conecta `Pedido` e `Produto` na relação **N:N**: um pedido pode ter vários produtos, e um produto pode estar em vários pedidos.

Ela tem duas chaves estrangeiras (`pedido_id` e `produto_id`) e dois campos próprios (`quantidade` e `preco_unit`). Guardar o `preco_unit` aqui é importante: o preço do produto pode mudar no futuro, mas o que o cliente pagou deve permanecer registrado.

Ao final, `Base.metadata.create_all(engine)` envia todos os `CREATE TABLE` de uma vez para o banco.

```python
# Modelo ORM para a tabela associativa de itens do pedido (relação N:N)
class ItemPedido(Base):
    __tablename__ = "tb_itens_pedidos"

    id:         Mapped[int]     = mapped_column(primary_key=True)
    pedido_id:  Mapped[int]     = mapped_column(ForeignKey("tb_pedidos.id"),  nullable=False)
    produto_id: Mapped[int]     = mapped_column(ForeignKey("tb_produtos.id"), nullable=False)
    quantidade: Mapped[int]     = mapped_column(Integer,        nullable=False)
    preco_venda: Mapped[Decimal] = mapped_column(Numeric(10, 2), nullable=False)

    # Relacionamentos: acessa pedido e produto diretamente como atributos
    pedido:  Mapped[Pedido]  = relationship(back_populates="itens")
    produto: Mapped[Produto] = relationship(back_populates="itens")

    def __repr__(self):
        return f"ItemPedido(pedido={self.pedido_id}, produto={self.produto_id}, qtd={self.quantidade})"


# Sincroniza todos os modelos com o banco (cria tabelas que ainda não existem)
Base.metadata.create_all(engine)
print("Modelo ItemPedido definido ✅")
print("\nTodas as tabelas sincronizadas com o banco! ✅")
```

---

**[4:00 – 7:00] O que o `relationship` faz na prática**

O que o ORM fez aqui?

Depois de definir os quatro modelos, vale parar um momento para entender o que o SQLAlchemy registrou internamente:

```
Classe Python          →  Tabela no banco
Atributo Mapped[str]   →  Coluna VARCHAR
Atributo Mapped[int]   →  Coluna INTEGER
relationship(...)      →  Navegação entre tabelas sem JOIN manual
```

O `relationship` é especialmente poderoso porque cria atalhos nos dois sentidos:

```python
cliente.pedidos           # lista todos os pedidos do cliente
pedido.cliente.nome       # acessa o nome do cliente a partir de um pedido
pedido.itens[0].produto   # acessa o produto do primeiro item do pedido
```

Nenhuma dessas navegações exige que você escreva SQL, o ORM monta o JOIN automaticamente quando você acessa o atributo.

---

**[7:00 – 10:30] A Session — o carrinho de transações**

A `Session` é como um **carrinho de transações**:  
Podemos adicionar objetos, modificar, e só envia tudo ao banco quando chama `.commit()`.

```
session.add(objeto)  →  objeto entra no "carrinho" (só em memória)
session.flush()      →  envia ao banco mas não confirma (útil para obter IDs)
session.commit()     →  confirma a transação: sem volta!
session.rollback()   →  desfaz tudo desde o último commit
```

O trecho abaixo mostra o fluxo mais simples: criar um objeto, adicioná-lo à session e commitar.  
Repare que o `id` só existe **depois** do `commit()` é o banco que gera esse valor.

Agora vamos adicionar informações:

```python
# Sessão: contexto de transação para operações com o banco
# with garante que a sessão seja fechada mesmo com erros
with Session(engine) as session:

    # Cria uma instância do modelo Cliente (ainda não foi salva)
    novo_cliente = Cliente(
        nome="Deborah Foroni",
        email="deborah.foroni@datavendas.com",
        cidade="São Paulo",
        estado="SP",
        data_cadastro=datetime.now(),
    )

    # Adiciona o cliente à sessão (stage: "pending")
    session.add(novo_cliente)
    # Commit: envia a transação ao banco (stage: "persistent")
    session.commit()

    print(f" Objeto: {novo_cliente}")

    # Após commit, o SQLAlchemy atribui o ID gerado pelo banco automaticamente  
    print(f" Cliente inserido! ID gerado pelo banco: {novo_cliente.id}") 
```

---

**[10:30 – 13:00] Inserindo com relacionamentos e `flush()`**

Agora vamos inserir com relacionamentos:

Uma das grandes vantagens do ORM é que você **não precisa informar chaves estrangeiras manualmente**. Basta passar o objeto relacionado, o SQLAlchemy resolve o ID sozinho.

Neste exemplo vamos criar em sequência:

1. Um `Produto` → `session.flush()` para obter o ID sem commitar ainda
2. Um `Pedido` passando `cliente=deborah` em vez de `cliente_id=deborah.id`
3. Um `ItemPedido` ligando pedido e produto via objetos, não via IDs

```
session.flush()
  ↓
Envia o INSERT ao banco dentro da transação atual
Gera o ID do objeto (sem commit)
Permite usar esse ID em objetos dependentes
  ↓
session.commit()
  ↓
Confirma tudo de uma vez
```

> 💡 Use `flush()` quando precisar do ID de um objeto para criar outro que depende dele sem precisar commitar antes.

```python
# Inserindo dados relacionados: produto, pedido e item
with Session(engine) as session:

    # Cria uma instância de produto
    notebook = Produto(
        nome="Notebook Pro 15",
        categoria="Eletrônicos",
        preco_atual=Decimal("3499.90"),
        ativo=True,
    )
    session.add(notebook)
    # flush: gera o ID do produto sem fazer commit (stage intermediário)
    session.flush()

    # Query para buscar o cliente pela sua característica única (email)
    from sqlalchemy import select
    # scalars: retorna apenas os objetos (não tuplas)
    deborah = session.scalars(select(Cliente).where(Cliente.email == "deborah.foroni@datavendas.com")).first()

    # Cria um pedido associado ao cliente
    # Note: passamos o objeto cliente, não o ID
    # O ORM automaticamente extrai o cliente_id
    pedido = Pedido(
        cliente=deborah,
        data_pedido=datetime.now(),
        valor_total=Decimal("3499.90"),
        status="Criado",
    )
    session.add(pedido)
    session.flush()  # gera o ID do pedido

    # Cria um item associando o pedido e produto
    item = ItemPedido(
        pedido=pedido,
        produto=notebook,
        quantidade=1,
        preco_venda=notebook.preco_atual,
    )

    session.add(item)    
    print(f"Item: {item.quantidade}x {item.produto.nome} @ R${item.preco_venda}")
    print(f"Pedido {pedido.id}: criado para {pedido.cliente.nome}")
    
    # Commit final: salva tudo de uma só vez
  
    session.commit()    
```
---

**[13:00 – 15:00] Rollback na prática**

Em pipelines de dados, erros acontecem, temos muitas vezes email duplicados, valor nulo onde não pode, chave estrangeira inexistente.  

Se um `commit()` falhar no meio de várias operações, o banco pode ficar em estado inconsistente.

A solução é sempre envolver operações críticas em `try/except` e chamar `session.rollback()` no bloco de erro:

```
try:
    session.add(...)
    session.commit()     ← se falhar aqui...
except Exception:
    session.rollback()   ← ...tudo é desfeito, banco volta ao estado anterior
```

No exemplo abaixo, tentamos inserir um cliente com um e-mail que já existe no banco.  
A constraint `unique=True` do campo `email` vai rejeitar a operação e o rollback garante que nada parcial fique salvo.

```python
# Demonstrando transações: commit e rollback
with Session(engine) as session:
    try:
        # Tentando inserir um cliente com email duplicado
        # Isso violará a constraint unique=True do email
        cliente_duplicado = Cliente(
            nome="Deborah Foroni",
            email="deborah.foroni@datavendas.com",
            cidade="São Paulo",
            estado="SP",
            data_cadastro=datetime.now(),
        )
        session.add(cliente_duplicado)
        # Commit vai lançar uma exceção aqui
        session.commit()
        print("Inserido (não deveria chegar aqui)")

    except Exception as e:
        session.rollback()  # desfaz tudo!
        print(f"❌ Erro tratado: {type(e).__name__}")
        print(f"   Email duplicado detectado banco permanece íntegro")
```
---

**Exercício com IA**

Agora precisamos criar popular nosso banco de dados e para isso vamos pedir ajuda para a IA através desse prompt.

**Prompt para gerar dados de teste:**
```
Crie um script Python que popula o banco com dados fictícios realistas
usando os modelos SQLAlchemy abaixo. Gere 20 clientes brasileiros com
cidades e estados reais, 50 produtos em 5 categorias, e 100 pedidos
distribuídos aleatoriamente. Use a biblioteca Faker para os dados.

[cole os modelos aqui]
```
---