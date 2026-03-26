## Aula 01 — Setup e Conexão com o Banco

**Duração estimada:** 12 min

---

**[0:00 – 1:30] Gancho: o problema que o SQLAlchemy resolve**

Você acabou de entrar na DataVendas como analista de dados. Primeiro dia, primeiro desafio: conectar o banco que vai sustentar todos os relatórios da empresa.

Você poderia montar strings SQL manualmente — mas isso é frágil, inseguro e difícil de migrar entre bancos. 

O SQLAlchemy resolve isso: ele traduz código Python para SQL de forma automática e segura.

Mostre o diagrama mental:
```
Seu código Python → SQLAlchemy → SQL → Banco de dados
```

O benefício prático de usar essa biblioteca é: trocar de SQLite para PostgreSQL em produção é uma linha de código.

---

**[1:30 – 3:00] Instalação e verificação do ambiente**

Se você não tiver usando o ambiente Poetry detalhes no arquivo readme, execute o `pip install sqlalchemy pandas`, depois rode a célula de verificação de versão. Confirme que o ambiente está pronto com o print de saída.

```python
# Bibliotecas necessárias para a aula
from pathlib import Path
import sqlalchemy
from sqlalchemy import create_engine
print(f"SQLAlchemy versão: {sqlalchemy.__version__}")
print("Ambiente pronto! ✅")
```

---

**[3:00 – 7:00] Criando a Engine**

A `Engine` é o ponto de entrada do SQLAlchemy. Ela representa a **conexão com o banco** e gerencia o pool de conexões por baixo dos panos.

Vamos agora criar a URL de conexão que por padrão tem: 
```
dialeto+driver://usuario:senha@host:porta/banco
```

Vamos criar a conexão usando a função create_engine:

```python
# Cria o diretório se não existir
Path("../database").mkdir(exist_ok=True)

# Criando a engine, a "ponte" com o banco
engine = create_engine(
    "sqlite:///../database/datavendas.db",
    echo=True # Mude para True para ver o SQL gerado no console, isso é ótimo para aprender
)

print(f"Engine criada: {engine}")
```

Obvserve o parâmetro `echo=True` — ele é um ótimo aliado para aprender porque mostra o SQL gerado no console.

Outro detalhe importante é que em projetos reais, a URL de conexão **nunca** fica hardcoded no código.  

Ela é carregada por variáveis de ambiente para não expor credenciais no repositório.
Fica igual esse exemplo aqui.

---

**[7:00 – 10:00] Testando a conexão**

Agora iremos, testar a nossa conexão:

```python
# Vamos importar o módulo necessário para executar as queries SQL
from sqlalchemy import text

# Abrindo uma conexão e executando uma query simples
with engine.connect() as conn:
    resultado = conn.execute(text("SELECT 'Conexão OK' AS status"))
    print(resultado.fetchone()[0])

# O 'with' garante que a conexão seja fechada automaticamente: boas práticas

```

O `with` é uma boa prática: ele garante que a conexão seja fechada mesmo em caso de erro.

Vamos observar o log gerado pelo `echo=True`, aqui vemos o que foi feito pela biblioteca.

---

**[10:00 – 12:00] As duas camadas do SQLAlchemy**

A biblioteca SQLAlchemy tem duas camadas: **Core** (SQL mais próximo do banco) e **ORM** (objetos Python). Durante o curso vamos usar as duas camadas.

Agora vamos verificar se o banco foi criado para isso vamos utilizar o módulo de inspeção para verificar que o banco foi criado e está vazio, pronto para ser usado na próxima aula.

```python
# Vamos importar o módulo inspect do SQLAlchemy para introspecção do banco
from sqlalchemy import inspect

# Vamos criar um inspetor para o engine, permitindo examinar a estrutura do banco
inspector = inspect(engine)

# Agora, obter a lista de nomes das tabelas existentes no banco de dados
tabelas = inspector.get_table_names()

# Verifica se há tabelas no banco
if tabelas:
    print("Tabelas no banco:")
    # Itera sobre a lista de tabelas e imprime cada uma
    for t in tabelas:
        print(f"  - {t}")
else:
    # Se não há tabelas, informa que o banco está vazio e pronto
    print("Banco criado e vazio pronto para a próxima aula!")
```

Agora vamos usar IA para resolver o nosso exercício: vamos ver o prompt sugerido para gerar a string de conexão PostgreSQL. Neste exemplo, estou usando o Copilot

```
Gere o código SQLAlchemy para criar uma engine conectando ao PostgreSQL.
Host: localhost, porta: 5432, banco: datavendas, usuário e senha via variável de ambiente.
Inclua tratamento de erro de conexão e use o padrão com .env.
```

Um ponto importante  **nunca colar credenciais reais** em prompts, isso pode gerar probelmas de pessoas não autorizadas utilizar as credenciais e acessar informações privadas.

---