# Transcrição da sessão de IA

**Pergunta**

> Estamos estudando sobre Bridge Tables, Junk Dimensions e SCD aplicados ao projeto que estamos desenvolvendo para a Sindusfarma.
> Identifique qual cubo que modelamos na Sprint 1 (disponíveis no Data Model Canvas, já preenchido) seria adequado para implementarmos esses conceitos.
>
> Para isso, o passo a passo seria o seguinte:
> 1. Reler o canvas, retomando como referência os cubos, o grão, as dimnesões e a estrutura
> 2. Encontrar a relação N:N: Identificar um relacionamento N para N adequado para a bridge table e alguma dimensão.

**Resposta**

Reli o canvas (`docs/dmc.md` + `docs/dmc.json`) e cruzei com os perfis apurados em `docs/dados/`. Segue o resultado dos dois passos.

Passo 1, o que o canvas fixa hoje:

| | Cubo 1 Ciclo de Vida | Cubo 2 Participação | Cubo 3 Resultado | Cubo 4 Qualidade |
|---|---|---|---|---|
| Grão | 1 pesquisa (366) | 1 participação pessoa×pesquisa (18.735) | 1 opção marcada por respondente×pergunta (20.607) | 1 apuração×entidade×atributo×regra |
| Tipo de fato | accumulating snapshot | transaction | transaction | periodic snapshot |
| Dimensões | Tempo, Pesquisa, Área, Situação, Tipo, Perfil | Tempo, Pesquisa, Empresa, Pessoa | Tempo, Pesquisa, Pergunta, Alternativa, Empresa | Tempo, Entidade, Atributo, Regra |
| Junk hoje | DIM_PERFIL_PESQUISA | vazio | vazio | vazio |
| Bridge hoje | vazio | vazio (descartada explicitamente) | vazio | vazio |
| SCD hoje | vazio | vazio | vazio | vazio |

No `dmc.json` os campos `bridgeTables`, `junkDimensions` e `lists/scd` existem e estão em branco em todos os cubos, exceto a junk do Cubo 1. Ou seja, o exercício tem lugar reservado no entregável.

Passo 2, onde está a relação N:N. Apliquei o teste de cardinalidade cubo a cubo, com contagem e não com impressão:

| Cubo | Candidato | Verificação | Veredito |
|---|---|---|---|
| 1 | Pesquisa e Área | frequências de `area` somam exatamente 366 (152+88+44+...+1). Mesma coisa em `tipo` (170+84+...+1 = 366) | 1:N. Sem bridge |
| 2 | Pessoa e Empresa | `clientes.id_empresa` é coluna única por cliente, 5.429 clientes para 410 empresas | 1:N. O canvas já descartou, e com razão |
| 2 | Empresa e Pesquisa | é N:N, mas quem já resolve isso é a própria FATO_PARTICIPACAO. Tabela fato não é bridge | Não se aplica |
| 4 | Entidade e Atributo | metadado hierárquico, 1 atributo pertence a 1 entidade | 1:N. Sem bridge |
| 3 | **Pergunta e Alternativa** | `questoes.alternativas` é campo multivalorado com separador ` \| `. A lista `"Não \| Sim"` aparece em 31 questões distintas (13,4%) | **N:N confirmado** |
| 3 | **Resposta e Alternativas marcadas** | 80 das 231 questões são `caixas_de_selecao` (34,6%). O par respondente+pergunta gera 9.543 combinações para 20.607 linhas | **N:N confirmado** |

O cubo é o **Cubo 3, Resultado da Pesquisa**. E ele não tem um N:N, tem dois, em níveis diferentes.

No nível do instrumento, Pergunta e Alternativa. Uma pergunta oferece N alternativas; uma alternativa canônica ("Sim", "Não", "Presidente", "Outro") é oferecida por N perguntas. Nenhum dos dois lados pode carregar a chave estrangeira, que é a definição de bridge. A evidência dura: `alternativas` tem 116 strings de lista distintas para 231 questões, e o dado de resposta mostra 246 textos atômicos distintos, com "Presidente" aparecendo 992 vezes em questões diferentes.

```
DIM_PERGUNTA ──1:N──> PONTE_PERGUNTA_ALTERNATIVA <──N:1── DIM_ALTERNATIVA
                      (sk_pergunta, sk_alternativa,
                       ordem_exibicao, flag_exclusiva)
```

No nível do fato, Grupo de marcações e Alternativa, que é o caso canônico de Kimball. Este é o mais forte, porque resolve um problema que o canvas já documentou como defeito: numa questão de caixas de seleção, um único evento de resposta produz várias linhas, e por isso `qtd_marcacao` não é a contagem de respostas. O canvas registra o erro de 2,2 vezes que isso causaria. É exatamente o padrão de dimensão multivalorada, o "grupo de diagnósticos por paciente" do Kimball:

```
FATO_RESPOSTA (grão elevado: 1 linha por respondente×pergunta = 9.543)
   └─ sk_grupo_marcacao ──> DIM_GRUPO_MARCACAO
                                  └──> PONTE_GRUPO_MARCACAO ──> DIM_ALTERNATIVA
                                       (sk_grupo, sk_alternativa,
                                        fator_ponderacao, ordem)
```

O ganho é concreto e mensurável:

| | Modelo atual | Com a bridge |
|---|---|---|
| Linhas do fato | 20.607 | 9.543 |
| `qtd_resposta` aditiva? | não (conta marcações) | sim |
| PN12 "distribuição das respostas" | denominador precisa ser declarado à mão | sai do `fator_ponderacao` |
| Leitura de impacto ("quais opções foram citadas") | ok | ok, join sem peso |

O `fator_ponderacao` é 1/n para caixas de seleção (a soma do grupo fecha em 1, leitura ponderada correta) e 1 para as células das 3 questões `matriz_avaliacao`, onde cada célula é um julgamento binário independente e o denominador natural é respondentes × linhas. Essa distinção precisa ficar declarada no atributo `tipo_ponderacao` do grupo, senão a matriz que chega a 96 linhas para o mesmo par volta a distorcer o percentual.

O mesmo cubo também comporta junk e SCD sem forçar nada. A junk seria DIM_PERFIL_QUESTAO, com `tipo` (5) × `confianca` (2) × `formato_arquivo` (2), no máximo 20 linhas: atributos de baixa cardinalidade, sem dimensão natural de origem, e dois deles são metadado de extração, que é o caso de manual da junk. Simétrico ao que o Cubo 1 já fez com DIM_PERFIL_PESQUISA.

O SCD seria DIM_PERGUNTA como Tipo 2. 84 pesquisas são "Pesquisa fixa anual" e 4 são "fixa bianual", ou seja, 24% do acervo repete instrumento entre edições. O `enunciado` tem 176 valores distintos para 231 questões, e vários aparecem exatamente 2 vezes em pesquisas diferentes: é a mesma pergunta reaplicada, com reformulação de texto. Tipo 2 preserva a redação da época e permite comparar edições pela chave durável. Já DIM_ALTERNATIVA pede Tipo 1, porque correção de grafia não merece versão nova, e DIM_EMPRESA, conformada com o Cubo 2, pede Tipo 2 em `Associado`, `Tipo de Associado` e `% de mercado`.

Um alerta de honestidade: a seção 4.6 do canvas declara que nenhuma base traz data de vigência. Isso não invalida o SCD, mas define o ponto de partida. A versão 1 de cada linha nasce na implantação, e o histórico só existe daí em diante. Prometer histórico retroativo continuaria sendo falso.

Duas coisas a apurar antes de fechar. As bases do parceiro não estão versionadas (`DADOS_PARCEIRO` não está definida neste ambiente), então dois números que sustentam o desenho ainda estão por contar: quantas alternativas atômicas distintas existem depois de explodir o campo por ` | `, que é o número de linhas de DIM_ALTERNATIVA; e a confirmação de que os enunciados repetidos pertencem a pesquisas diferentes, e não a duplicata dentro da mesma pesquisa, que é o que separa SCD Tipo 2 legítimo de problema de qualidade.

Quer que eu escreva esse script de apuração, ou que eu já redija a seção 4.3 revisada do canvas com bridge, junk e SCD preenchidos nos campos correspondentes do `dmc.json`?

---

**Pergunta**

> Gostei da proposta do cubo e da justificativa apresentada.
> 1. Pode iniciar a modelagem da Bridge Table: nome, colunas, foreign keys e, se alguma métrica aditiva for somada através dela, o peso.
> 2. Encontre atributos de baixa cardinalidade, já pensando em junk dimensions.
> 3. Escolha o SCD por dimensão
> Tudo isso pode fazer em um novo arquivo MarkDown.

**Resposta**

Arquivo criado: `docs/dimensoes_especiais_bridge_junk_e_scd.md`, com 347 linhas e 7 seções. Resumo do que ficou modelado.

Sobre a bridge table, são duas pontes, e só uma carrega peso.

A `PONTE_GRUPO_MARCACAO` é a bridge de agregação, com PK composta `(sk_grupo_marcacao, sk_alternativa)` e FKs para `DIM_GRUPO_MARCACAO` e `DIM_ALTERNATIVA`, mais as colunas de apoio `ordem_no_grupo`, `fator_ponderacao` e `tipo_ponderacao`. Ela exige uma `DIM_GRUPO_MARCACAO` nova, com `hash_grupo` (SHA-256 do conjunto ordenado) como chave natural e `qtd_alternativas_no_grupo` como o n do denominador. O grão do fato sobe de 20.607 para 9.543 linhas, o que devolve aditividade a `qtd_resposta`, que hoje não tem, e move a explosão para a ponte.

Os pesos, por tipo de questão: `multipla_escolha` (123) fator 1; `caixas_de_selecao` (80) fator 1/n, com o grupo somando 1; `matriz_avaliacao` (3) fator 1, com o grupo somando n porque cada célula é julgamento independente; numérica e aberta (25) em grupo sentinela de um membro.

A regra crítica que documentei é que `qtd_alternativas_marcadas` não pode atravessar a ponte, porque vira n². Na maior questão de matriz do acervo, com `qtd_alternativas = 96`, uma única resposta devolveria 9.216 em vez de 96.

A `PONTE_PERGUNTA_ALTERNATIVA` é ponte de catálogo. Resolve o N:N do instrumento e responde quais alternativas foram oferecidas e nunca escolhidas. Não carrega peso, e documentei a proibição de somar por ela, com o fator correto (`1/qtd_alternativas_oferecidas`) caso alguém precise no futuro.

Sobre as junk dimensions, levantei todas as 17 colunas com até 20 valores distintos e classifiquei o destino de cada uma. Duas junks novas: `DIM_PERFIL_QUESTAO`, com `tipo` (5) × `confianca` (2) × `formato_arquivo` (2) × faixa de alternativas × flag de múltipla marcação, cartesiano de 200 e real abaixo de 30; e `DIM_PERFIL_RESPOSTA`, com 5 flags do evento (matriz 19,4%, valor livre 59,6%, sem marcação 49,8%, empresa identificada), no máximo 32 linhas.

Incluí a tabela de candidatos recusados com o motivo, porque é onde o conceito costuma ser mal aplicado: `Nacionalidade`, `Associado` e `Tipo de Associado` têm casa natural em `DIM_EMPRESA`, e `dia da semana` é derivado e pertence a `DIM_TEMPO`. O argumento que amarra junk e SCD é que, como `DIM_PERGUNTA` vai a Tipo 2, deixar `confianca` e `formato_arquivo` dentro dela faria metadado de extração versionar a cada reformulação de enunciado.

Sobre o SCD, montei a tabela de decisão para as 16 dimensões, 13 do canvas mais 3 novas: Tipo 0 em 5, Tipo 1 em 6 e Tipo 2 em 5, com `DIM_EMPRESA` e `DIM_PESSOA` em modo híbrido.

Três pontos exigem decisão antes de implementar. O `% de mercado` fica fora do Tipo 2 em `DIM_EMPRESA`, porque são 222 valores distintos, versionar isso explode a dimensão, e o valor já vive datado como métrica no Cubo 2. A `DIM_PERGUNTA` não tem chave durável, já que `id_pergunta` é único por linha (231 para 231), então o Tipo 2 depende de derivar `id_pergunta_duravel` por hash do enunciado normalizado, restrito à família de pesquisa recorrente; até validar, opera como Tipo 1. E a `DIM_REGRA_QUALIDADE` é Tipo 2 obrigatório, porque `score_saude_dado` é média ponderada e alterar o peso no lugar reescreve a série inteira.

Também documentei a regra de ordem de carga, com a normalização Tipo 1 antes da comparação Tipo 2, senão as 26 variantes excedentes apuradas viram versões históricas falsas, e o fato de que a série do Tipo 2 só começa na implantação, coerente com a Seção 4.6 do canvas.

O arquivo está untracked. Quer que eu commite e depois aplique as mudanças no `dmc.json` e na Seção 4.3 do `dmc.md`? A Seção 6 do documento já traz o mapeamento campo a campo.

---

**Pergunta**

> Excelente, gostei das decisões.
>
> Inclua no documento um diagrama visual (Mermaid) da nossa solução — por exemplo, o esquema da bridge table com a fato e a dimensão, ou a comparação entre as estratégias de SCD escolhidas.
>
> Pode enxugar esse documento, está muito longo.

**Resposta**

Documento reescrito de 347 para 254 linhas, cerca de 27% menor, com três diagramas Mermaid.

O primeiro é um `erDiagram` do Cubo 3, na Seção 2.1, e é o principal. Mostra o esquema completo com as duas pontes resolvendo os N:N, as colunas de cada tabela e a cardinalidade explícita. Dá para ler direto no diagrama que `DIM_GRUPO_MARCACAO ||--|{ PONTE_GRUPO_MARCACAO }o--|| DIM_ALTERNATIVA` é o par que quebra o muitos para muitos, e os comentários carregam as regras: `fator_ponderacao` marcado como "O PESO" e `qtd_alternativas_marcadas` como "NAO atravessa a ponte".

O segundo é um `flowchart LR` do peso, na Seção 2.2, mostrando o fan-out de uma resposta com 3 alternativas marcadas e terminando nas duas leituras: sem peso soma 3, que é a leitura de impacto, e com peso soma 1, que é a distribuição. É o conceito que costuma ser o mais difícil de explicar em texto.

O terceiro é um `flowchart TD` da decisão de SCD, na Seção 4, uma árvore com as 16 dimensões distribuídas nas folhas. Serve como a comparação entre as estratégias que você sugeriu: dá para ver de relance por que `DIM_TEMPO` e as junks são Tipo 0, por que `DIM_PERGUNTA` cai no ramo "sem chave durável" e por que só 4 dimensões chegam ao Tipo 2.

O corte incidiu sobre prosa redundante. As tabelas de colunas, os pesos e a tabela de SCD por dimensão ficaram integrais, porque são o entregável. A tabela de 17 colunas de baixa cardinalidade virou texto corrido nas duas junks, já que as informações de destino se repetiam na tabela de SCD; as seções 3.2 a 3.5 viraram o próprio `erDiagram` mais dois parágrafos; e as seções 6 e 7 antigas viraram uma só.

Testei três pontos de sintaxe que quebrariam a renderização no GitHub. Troquei `PK_FK`, que não é chave válida em Mermaid, por `FK` com o comentário "PK composta". Removi as tags `<b>`, porque o GitHub renderiza com `htmlLabels` desabilitado e elas apareceriam literais. E tirei os `---` que usei como separador dentro dos nós, já que colidem com o token de aresta do parser. Também quebrei o encadeamento `F --> P --> A & B & C` em duas linhas.

Não consegui renderizar os diagramas localmente para confirmar visualmente, porque não há Mermaid CLI no ambiente, então a validação foi de sintaxe e não de resultado renderizado. Vale conferir a aparência no GitHub depois do commit.

---

**Pergunta**

> Gere uma transcrição desta conversa e adicione a C:\Users\Caleb\GitHub\ponderada-datawarehouse-iii\transcricao-ia.md

**Resposta**

Transcrição criada em `transcricao-ia.md`, com cabeçalho de metadados, um bloco por turno com prompt, ferramentas executadas, achados de dados e resposta, e uma seção final de observações sobre o uso da IA. Reproduzi as tabelas de evidência porque são o que mostra o raciocínio, mas remeti a `resposta.md` no lugar de duplicar o documento final. Incluí também os passos que não deram certo, porque uma transcrição que só mostra o caminho limpo perderia valor como registro.

---

**Pergunta**

> Quero só a transcrição pura: perguntas e as respostas, não precisa das tools e bashes.

**Resposta**

Reescrevi o arquivo mantendo apenas os turnos de pergunta e resposta, sem as tabelas de ferramentas, os comandos executados e a seção de observações sobre o processo. As respostas ficaram mais completas do que na versão anterior, já que agora são o conteúdo inteiro do arquivo.

---

**Pergunta**

> Escreva mais simples: Não precisa repartir os turnos e dados adicionais - só as perguntas e as respostas.

**Resposta**

Simplifiquei o formato: saíram os títulos de turno, o cabeçalho de metadados e os subtítulos internos das respostas. Ficaram apenas os pares de pergunta e resposta, separados por linha horizontal.
