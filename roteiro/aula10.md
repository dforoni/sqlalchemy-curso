## Aula 10 — Views e Triggers: Lógica que Vive no Banco

**Duração estimada:** 14 min

---

**[0:00 – 1:30] Gancho: o mesmo JOIN em 10 notebooks**

O time de analytics repetiu o mesmo JOIN quádruplo em 10 notebooks. Quando uma coluna foi renomeada, quebraram os 10 de uma vez. Existe uma solução mais inteligente: encapsular o JOIN em uma **View**.

---

**[1:30 – 3:30] O que é uma View e qual problema ela resolve**

Mostre o antes e o depois:
```
Sem view: cada notebook repete JOIN clientes + pedidos + itens + produtos
Com view: SELECT * FROM vw_vendas_detalhadas  ←  lógica em um só lugar
```

Alterando a lógica em um único lugar, todos os notebooks se beneficiam. Views são como funções reutilizáveis no banco.

---

**[3:30 – 6:30] Criando a view com `text()`**

Explique que o SQLAlchemy não tem método ORM dedicado para criar Views — isso é DDL, então usamos SQL nativo via `text()`. Execute a criação da `vw_vendas_detalhadas`.

Avise: views criam objetos permanentes no banco. Em ambiente de desenvolvimento, faça uma cópia do banco antes de experimentar.

---

**[6:30 – 9:30] Consultando views: 3 formas**

Percorra as três formas com exemplos executados:

1. `text()` — mais simples, ótimo para Pandas
2. `Table(autoload_with=engine)` — lê a estrutura do banco e cria objeto Python com `.c`
3. Classe ORM mapeada — acesso por atributo, integração com outros modelos

Use a tabela do notebook para recomendar: comece com `Table(autoload_with=)`. Use a classe ORM só se precisar de métodos customizados.

Reforce: views são **somente leitura** — escreva sempre nas tabelas originais.

---

**[9:30 – 11:30] Triggers: o problema que eles resolvem**

`valor_total` em `tb_pedidos` depende do código Python lembrar de recalcular. Se alguém inserir um item direto no banco, o campo fica desatualizado silenciosamente.

Mostre o diagrama antes/depois do trigger do notebook.

---

**[11:30 – 13:30] Criando os triggers e testando**

Execute a criação dos três triggers (INSERT, UPDATE, DELETE em `tb_itens_pedidos`). Mostre a anatomia de um trigger: `AFTER INSERT`, `NEW`, `BEGIN/END`.

Insira um novo item e mostre que `valor_total` é atualizado automaticamente. Explique o `session.refresh()` — o trigger rodou no banco, mas o objeto Python ainda tem o valor antigo em memória; o `refresh()` sincroniza.

---

**[13:30 – 14:00] Fechamento**

Feche com o prompt de IA para gerar uma view de receita mensal. Recapitule: Views eliminam duplicação de lógica; Triggers garantem consistência sem depender do código da aplicação.

---