## Aula 04 — SQL Injection e Segurança

**Duração estimada:** 13 min

---

**[0:00 – 1:30] Gancho: dados chegando de fora**

Nesta aula vamos aprender:

- Entender o que é SQL Injection e por que é perigoso
- Ver o problema acontecendo ao vivo (de forma controlada)
- Aprender a prevenir com consultas parametrizadas
- Criar funções seguras de busca para o seu portfólio

Bom, o nosso gerente pediu uma busca de clientes por ID via API externa. O tech lead nos chama antes e diz: *"Cuidado com SQL Injection, já vi sistema cair por isso."*

O SQL Injection acontece quando dados externos entram diretamente no texto da query. O banco não distingue lógica de dado e executa tudo.

É o tipo de vulnerabilidade mais comum em sistemas com banco de dados, e aparece em vagas de emprego frequentemente: **todo profissional de dados precisa conhecer e evitar**.

Vamos importar e criar a conexão com o nosso banco de dados.

---

**[1:30 – 4:30] Demonstrando o problema ao vivo**

Vamos imaginar que construimos uma função de busca assim:

```python
# CÓDIGO VULNERÁVEL — serve apenas para demonstrar o problema!

def buscar_cliente_vulneravel(engine, id_input):
    """NUNCA faça isso em produção!"""
    # Aqui o valor externo entra diretamente na string SQL
    # Se id_input for malicioso, vira comando SQL injetado.
    sql = f"SELECT id, nome, email FROM tb_clientes WHERE id = {id_input}"
    print(f"SQL gerado: {sql}")
    
    with engine.connect() as conn:
        # Executa a query montada dinamicamente (vulnerável a injeção)
        return conn.execute(text(sql)).fetchall()


# Uso normal parece inocente
print("--- Busca normal ---")
resultado = buscar_cliente_vulneravel(engine, "1")
print(f"Resultado: {resultado}")
```

Mas agora com uma entrada maliciosa:

```python
# Agora com uma entrada maliciosa:
print("--- Ataque de injeção ---")
resultado_ataque = buscar_cliente_vulneravel(engine, "1 OR 1=1")
print(f"\nLinhas retornadas: {len(resultado_ataque)}")
print("Esperávamos 1 registro — mas a injeção 'OR 1=1' retornou a tabela INTEIRA!")
print()
print("Em produção, isso poderia expor dados sensíveis de todos os clientes.")
```
### O que aconteceu?

1. A Intenção Original
Normalmente, a função buscar_cliente_vulneravel esperaria um ID simples, como "1". O SQL gerado internamente seria algo como:

```sql
SELECT id, nome FROM tb_clientes WHERE id = 1
```

Resultado: O banco busca apenas o registro onde o ID é exatamente 1.

2. A "Malícia" da Injeção
Quando você passa a string "1 OR 1=1", você não está enviando apenas um dado, mas sim novo código SQL. Se a função apenas concatenar esse texto na query, o comando final vira:

```sql
SELECT id, nome FROM tb_clientes WHERE id = 1 OR 1=1
                                              ^^^^^^^^
                                   Isso é sempre TRUE!
                              Retorna todos os registros.
```

3. Por que retorna a tabela inteira?
     O banco de dados avalia a cláusula WHERE para cada linha da tabela:

     Ele verifica se id = 1 (isso pode ser verdadeiro ou falso).

     Depois, ele verifica se 1=1.

     Como 1=1 é sempre verdadeiro (True), a expressão completa (Falso OR Verdadeiro) resulta em Verdadeiro.

     Como o resultado é sempre positivo para todas as linhas, o banco ignora o filtro e entrega todos os dados existentes.
---

**[4:30 – 6:30] Por que o SQLAlchemy não resolve sozinho**

O simples uso do SQLAlchemy não protege se usarmos f-strings. 

O problema está em **como** você constrói a query, não em qual biblioteca usa. O mesmo problema existe com `pandas.read_sql()`, `psycopg2` — qualquer driver.

---

**[6:30 – 10:00] A solução: consultas parametrizadas**

A solução: consultas parametrizadas

A defesa correta é **separar código de dados**.  
O SQL vai com marcadores (`:parametro`) e os valores são enviados separadamente o banco nunca os mistura.


```python
# CÓDIGO SEGURO

def buscar_cliente_seguro(engine, id_input):
    """Versão segura: dados e código são separados."""
    
    # Marcador nomeado e não é concatenação!
    sql = text("SELECT id, nome, email FROM tb_clientes WHERE id = :id")
    #                                                              ^^^
    #                                          O banco recebe isso como dado, não como SQL
    
    with engine.connect() as conn:
        return conn.execute(sql, {"id": id_input}).fetchall()


# Mesmo com a entrada maliciosa — agora é tratada como texto literal
print("--- Mesma entrada maliciosa, código seguro ---")
resultado_seguro = buscar_cliente_seguro(engine, "1 OR 1=1")
print(f"Linhas retornadas: {len(resultado_seguro)}")
print("O 'OR 1=1' foi tratado como string literal, não como lógica SQL")
print("   Nenhum dado vazou.")
```

---

**[10:00 – 13:00] Validação de tipo como camada extra**

Validação de tipo de dado como camada extra

Parametrizar é obrigatório. Validar o tipo é uma **camada adicional de defesa** e deixa o código mais robusto.

```python
def buscar_cliente_robusto(engine, id_input) -> list | str:
    """
    Versão:
    1. Valida o tipo da entrada
    2. Usa consulta parametrizada
    3. Retorna mensagem amigável em caso de erro
    """
    # Camada 1: validação de tipo
    try:
        id_valido = int(id_input)
    except (ValueError, TypeError):
        return f"Entrada inválida: '{id_input}' não é um ID numérico."

    # Camada 2: consulta parametrizada
    with engine.connect() as conn:
        resultado = conn.execute(
            text("SELECT id, nome, email, cidade FROM tb_clientes WHERE id = :id"),
            {"id": id_valido}
        ).fetchall()

    if not resultado:
        return f"Cliente com ID {id_valido} não encontrado."

    return resultado


# Testando os três cenários
print("✅ ID válido:",        buscar_cliente_robusto(engine, "1"))
print("❌ Injeção:",          buscar_cliente_robusto(engine, "1 OR 1=1"))
print("❌ Texto qualquer:",   buscar_cliente_robusto(engine, "abc"))
print("🔍 ID inexistente:",   buscar_cliente_robusto(engine, "9999"))
```
---

**Exercício com IA**

Agora vamos utilizar a IA para ajudar a gente a converter um código legado que estava usando o f-strings.

**Prompt para converter código legado:**
```
Este código usa f-strings para montar queries SQL (vulnerável).
Reescreva usando SQLAlchemy com consultas parametrizadas e adicione
validação de tipo nas entradas externas:
```

```python
def buscar_cliente_vulneravel(engine, id_input):
    sql = f"SELECT id, nome, email FROM tb_clientes WHERE id = {id_input}"
    print(f"SQL gerado: {sql}")
    
    with engine.connect() as conn:
        return conn.execute(text(sql)).fetchall()
```
---