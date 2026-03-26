## Aula 11 — UPDATE, DELETE e Transações

**Duração estimada:** 15 min

---

**[0:00 – 1:30] Gancho: hora de modificar dados reais**

Três tarefas chegaram ao mesmo tempo: reajustar preços em lote, cancelar pedidos antigos, migrar 50.000 registros. Estas operações exigem mais cuidado: um `DELETE` mal filtrado apaga o que não devia.

---

**[1:30 – 5:00] UPDATE: duas formas**

**Via ORM** (poucos registros, lógica Python): carrega o objeto, altera o atributo, faz commit. Simples e legível.

**Via `update()` direto** (muitos registros): constrói e executa um `UPDATE` SQL sem instanciar nenhum objeto. Muito mais rápido para centenas ou milhares de registros.

Execute os dois e compare. Mostre o aviso crítico: **UPDATE sem WHERE é desastre**. Mostre `result.rowcount` para confirmar quantas linhas foram afetadas.

---

**[5:00 – 7:30] Atualização em lote com `bulk_update_mappings`**

Mostre o padrão para atualizar uma lista de registros em lote — envia uma lista de dicionários ao banco de forma eficiente. Útil para pipelines de ingestão com atualizações massivas.

---

**[7:30 – 10:00] DELETE com segurança**

Mostre a exclusão via ORM. Execute o erro clássico: tentar apagar um `Pedido` com `ItemPedido` dependentes. O banco bloqueia por integridade referencial.

Mostre a ordem correta: apagar filhos antes de apagar o pai. Reforce que `DELETE` remove informação permanentemente — sempre filtrar com cuidado e conferir o `rowcount`.

---

**[10:00 – 13:00] Transações: tudo ou nada**

Explique o conceito de atomicidade: ou todas as operações acontecem, ou nenhuma. Mostre os três padrões:

```
session.commit()         → confirma tudo
session.rollback()       → desfaz tudo
with session.begin():    → commit OU rollback automático
```

Execute o cenário de falha no meio de uma transação — mostre que o `rollback()` desfaz tudo e o banco volta ao estado consistente.

---

**[13:00 – 15:00] Fechamento do curso**

Feche com o prompt de IA para gerar um batch update seguro com reajuste percentual e teto máximo de preço.

Recapitule a jornada: do setup e conexão, passando pelo Core e ORM, segurança, consultas avançadas, até operações de escrita com transações. O aluno agora tem as ferramentas para construir pipelines de dados robustos com SQLAlchemy.