# US-01 — Detalhe do aluno: rastreabilidade e requisitos

**Questões 1 e 2** · issue `USER_STORIES.md:53` — endpoint que recebe o ID/Matrícula do aluno e devolve um JSON com seus dados demográficos e a matriz de peso/impacto do modelo de ML.
**Base:** `origin/unstable` @ `66b8e24`

# Q1 — Matriz de Rastreabilidade e Requisitos


## Matriz

(Professor, o erro será citado abaixo foi identificado com o auxílio da IA)

| Requisito | Critério de aceite | Implementação | Teste | Commit |
|---|---|---|---|---|
| **Sub-tarefa 1**<br>`USER_STORIES.md:53` | *"retorne um JSON contendo seus dados demográficos e o array de peso/impacto das variáveis processadas pelo modelo de Machine Learning"* | `get_student()` `students.py:110`<br>`fetch_record()` `database.py:66`<br>`_top_factors()` `ml.py:71` | `test_students_detail`<br>⚠️ **parcial** | `a537160` |
| **RF13**<br>`PRD.md:43` | *"devo visualizar as principais variáveis que compuseram a nota (ex: bolsa, notas do semestre) e o percentual probabilístico de evasão"* — `USER_STORIES.md:48` | `_feature_detail()` `students.py:32`<br>`StudentModal.tsx:12-48` | `test_students_detail` | `a537160` |
| **RF12**<br>`PRD.md:42` | *"quando clico na linha de um estudante específico, então uma visualização detalhada (modal ou painel lateral) deve se sobrepor à tela atual"* — `USER_STORIES.md:47` | `StudentsTable.tsx:310`<br>`StudentsPage.tsx:78` | ⚠️ **nenhum** | `2dc6a23` |
| **RN01**<br>`PRD.md:55` | *"sendo proibida a exibição de rótulos binários definitivos (como 'Evadido/Retido')"* — `USER_STORIES.md:48` | `classify_risk()` `ml.py:52-56` | `test_predict_success_shape` | `a537160` |
| **RN02**<br>`PRD.md:56` | *"alunos com dados incompletos ou anômalos para o modelo devem ser listados, mas sinalizados visualmente (ex: 'Risco Indisponível/Inconclusivo')"* | `_risco_item()` `students.py:24-29` | `test_risco_item_maps_inconclusive_to_indisponivel` | `a537160`, `368f910` |
| **RN03**<br>`PRD.md:57` | *"não deve existir nenhuma opção, botão ou link de compartilhamento externo para a internet pública"* — `USER_STORIES.md:49` | `StudentModal.tsx` (sem ações de share) | ⚠️ **nenhum** | `a537160` |


### O que a tela promete

Ao clicar num aluno, o painel mostra os dados dele e a lista das variáveis
que pesaram no risco (bolsa, notas do semestre...). É essa lista que permite
ao coordenador decidir uma intervenção.

### O que acontece se o modelo falhar

O modelo pode falhar ao montar a lista. Aí o sistema:

- devolve **200** (sucesso);
- mostra o número do risco normalmente;
- mostra a lista **vazia** — sem aviso, sem erro e sem log.

O resultado parece normal. É como um boletim com a nota e sem o comentário
do professor: o número existe, mas não diz o que fazer.

### CADEIA QUEBRADA - Por que ninguém notou

O teste que existe só confere a lista **quando ela vem preenchida**
(`test_api.py:191`):

```python
    if detail["fatores"]:
        assert {"feature", "efeito"} <= set(detail["fatores"][0])
```

Vindo vazia, o `if` não entra, a conferência é pulada e o teste dá **ok do
mesmo jeito**. Um sistema entregando só metade do que promete passaria.

E o trecho que produz a lista vazia não é testado por ninguém
(`students.py:120-123`):

```python
        try:
            fatores = run_prediction(data, request.app.state.artifacts)["fatores"]
        except Exception:
            fatores = []
```

### Defeito    

**É defeito.** O usuário recebe menos do que foi prometido e não sabe: o
campo `fatores` é obrigatório no contrato (`types.ts:39`) e a tela só
esconde a seção quando ele falta (`StudentModal.tsx:144`), em vez de avisar.
A falha também é silenciosa — nem usuário nem operador percebem. E sem a
lista do que pesou, o número de risco fica solto: o coordenador não tem como
agir, que é o objetivo da própria issue (`USER_STORIES.md:53`).

### O que fazer

1. Trocar o `if` do teste por uma verificação que **reprove** quando a lista
   vier vazia.
2. Criar um teste que simula o modelo quebrado, deixando escrito o
   comportamento nesse caso.

Duas alterações pequenas.

# Q2 — Classificação dos requisitos atendidos

O `PRD.md` classifica os requisitos em **três** categorias, e a diferença é o que a questão pede: **funcional** responde *o que o sistema faz*, **não funcional** responde *com que qualidade*, e **regra de negócio** responde *o que o domínio proíbe* — não é funcional nem não funcional.

| Categoria | Requisitos atendidos | Evidência |
|---|---|---|
| **Funcional**<br>*o que o sistema faz* | **RF12** `PRD.md:42`<br>**RF13** `PRD.md:43` | `StudentsTable.tsx:310` → `StudentsPage.tsx:78` → `StudentModal.tsx:7`<br>`_feature_detail()` monta nome, rótulo e valor de cada feature — `students.py:32-49` |
| **Não funcional**<br>*com que qualidade* | **RNF01** `PRD.md:47`<br>**RNF03** `PRD.md:49` | `predict` roda no servidor e o JSON sai com `features` e `fatores` já calculados — `students.py:12` + `ml.py:85`<br>`StudentModal` busca o detalhe por hook, em componente isolado — `StudentModal.tsx:3` → `queries.ts:53` → `api.ts:81` |
| **Regra de negócio**<br>*o que o domínio proíbe* | **RN01** `PRD.md:55`<br>**RN02** `PRD.md:56`<br>**RN03** `PRD.md:57` | `classify_risk()` devolve **faixa** + probabilidade, nunca veredito — `ml.py:52-56`<br>`_risco_item()` troca o resultado por "Indisponível" em vez de esconder a linha — `students.py:24-29`<br>busca por `href`, `window.open`, `share`, `Compartilhar` em `StudentModal.tsx`: **zero ocorrências** |

**RNF01 é o que esta issue atende por natureza:** o PRD pede que a inferência fique no servidor e chegue pronta na API (`PRD.md:47`) — é o que o endpoint faz. O navegador recebe o número e a lista de fatores, e nunca executa o modelo.

**Fora do escopo:** `RNF04` (`PRD.md:50`, menos de 2 s) trata de filtro, busca e ordenação da **listagem**, não do detalhe — e nenhuma parte do projeto mede tempo de resposta. `RNF02` (`PRD.md:48`) e `RNF05` (`PRD.md:51`) são herdados do projeto, sem relação com o endpoint.

**Conclusão:** 2 funcionais (RF12, RF13), 2 não funcionais (RNF01, RNF03) e 3 regras de negócio (RN01, RN02, RN03).




