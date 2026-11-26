# Contexto técnico: app Receitas e Rendimentos (Consinco → Qlik Cloud)

> Documento para quem vai manter ou evoluir o app, ou para dar contexto a uma nova sessão com um assistente de IA.
> O script completo está em `receitas-rendimentos.qvs`. O app, sem dados, está em `receitas-rendimentos.qvf`.

## 1. Ambiente

- **ERP:** Consinco, módulo Receitas e Rendimentos. Oracle, schema `CONSINCO`.
- **BI:** Qlik Cloud.
- **Conexão:** o script usa `LIB CONNECT TO 'ETL:Marketway_Consinco';`, que é o nome da conexão no ambiente original. Troque pelo nome da sua conexão Oracle (ver README).
- **Seções do script:** `Main` (SETs padrão) e `Modelo` (todo o resto). O script usa quebras de linha CRLF.
- **Visualizações:** organograma (`sn-org-chart`) e Sankey (`qlik-sankey-chart-ext`), do Visualization bundle nativo da Qlik. Sem extensões de terceiros.

## 2. Objetivos

1. Ver graficamente a dependência entre receitas, em que o componente de uma receita é o produto de outra, com expandir e retrair.
2. Dado um produto ou receita, ver quanto de cada **insumo folha** é usado.
3. Dado um insumo folha, ver quais receitas o usam e quanto.
4. Dada uma quantidade de matéria-prima, saber quanto produto final dá para fazer.
5. Ver quais setores dependem de quais, em qualquer profundidade, e por meio de quais receitas.
6. Listar as receitas com subreceitas e número de caminhos.

## 3. Tabelas de origem (Consinco)

| Tabela | Papel | Colunas usadas | Filtro de ativo |
|---|---|---|---|
| `MRL_RECEITARENDTO` | Cabeçalho | `SEQRECEITARENDTO`, `RECEITARENDTO`, `INDRECEITARENDTO` ('C' receita, 'N' rendimento), `SEQSETORPROD` | `STATUSRECRENDTO = 'A'` |
| `MRL_RRPRODUTOFINAL` | O que sai | `SEQRECEITARENDTO`, `SEQPRODUTO`, `QTDUNIDPRODUZIDA` | `STATUS = 'A'` |
| `MRL_RRCOMPONENTE` | O que entra | `SEQRECEITARENDTO`, `SEQPRODUTO`, `QTDUNIDUTILIZADA` | `STATUSRRCOMPONENTE = 'A'` |
| `MRL_SETORPROD` | Setor de produção | `SEQSETORPROD`, `DESCSETORPROD` | — |
| `MAP_PRODUTO` | Descrição do produto | `SEQPRODUTO`, `DESCCOMPLETA` | — |

Se o produto de um componente da receita Y é produto final da receita X, então Y depende de X. A estrutura é uma rede (DAG), não uma árvore. Para desenhá-la como árvore, os caminhos são "explodidos": cada ocorrência de um nó recebe o caminho completo como ID.

## 4. Modelo de dados (estrela unificada)

Não há ilhas. Todas as tabelas de fatos se ligam à dimensão **Receitas** só por `%SeqReceitaRaiz`, e a carga não gera chaves sintéticas.

```
   Produtos ── Arvore ─────────┐
 ArvoreNiveis ─┘               │
   Insumos ── ReceitaInsumo ───┤
             ReceitaContem ────┼── Receitas
               RedeArestas ────┤
              LigacaoSetor ────┘
```

Não se usa uma tabela de fatos concatenada porque, no Qlik, a seleção corta as linhas da própria tabela. Com nós e insumos juntos, selecionar um insumo apagaria o organograma. Na estrela, selecionar um insumo filtra **receitas inteiras**, e a árvore de cada uma continua completa.

| Tabela | Grão | Campos principais |
|---|---|---|
| `Receitas` | Uma linha por receita ativa ('C') | `[Receita]`, `[Receita Código]`, `[Receita Setor]`, `[Receita Setor Código]`, `[Receita Produto]`, `[Receita Produto Código]`, `[Receita Tipo Produto]` (Final/Intermediário), `[Receita Profundidade]`, `[Receita Qtd Cadastros Dependentes]`, `[Receita Cadastro Suspeito]`, `[Receita Self-Service]`, `[Receita Usa Outros Setores]`, `[Receita Qtd Setores na Árvore]` |
| `Arvore` | Um nó por linha, explodido no Oracle por CTE recursiva | `%NoID`, `%NoPaiID`, `%SeqProduto`, `[Produto Pai Código]`, `[Nível]`, `[Nível Cadastro]`, `[Origem Código]`, `[Origem Cadastro]`, `[Origem Setor]`, `[Tipo Nó]` (Receita/Insumo), `[Qtd Utilizada]`, `[Qtd Produzida]`, `[Fator Acumulado]`, `[Ramo Principal]`, `[Folha Explosão]`, `[Nó Folha]` |
| `Produtos` | Produtos que aparecem na árvore | `%SeqProduto`, `[Produto Código]`, `[Produto]` |
| `ArvoreNiveis` | Um nó por linha (1:1 com Arvore) | Saída do `Hierarchy()`: `[Nó]`, `Nó1..N`, `[Caminho]`, `[Profundidade Nó]` |
| `ReceitaInsumo` | Receita × insumo folha (lista de materiais) | `%SeqInsumo`, `[Qtd Insumo por Unidade]`, `[Insumo Caminhos]` |
| `Insumos` | Um insumo folha por linha | `%SeqInsumo`, `[Insumo Código]`, `[Insumo]` |
| `ReceitaContem` | Receita × qualquer produto da árvore | `[Contém Produto]`, `[Contém Produto Código]` |
| `RedeArestas` | Arestas do grafo de cada receita, sem explodir caminhos | Para um futuro gráfico de rede |
| `LigacaoSetor` | Receita consumidora × receita fornecedora × produto, em qualquer nível | `%LigacaoID`, `[Setor Consumidor]`, `[Setor Fornecedor]`, `[Lig Receita Consumidora]` e `[Lig Receita Consumidora Código]`, `[Lig Receita Fornecedora]` e `[Lig Receita Fornecedora Código]`, `[Lig Produto Fornecido]`, `[Lig Qtd Utilizada]` (só nas ligações diretas), `[Lig Distância]`, `[Ligação Alcance]` (Direta/Indireta), `[Ligação Tipo]` (Mesmo setor/Entre setores) |

**Ordem de grandeza** num cadastro com cerca de 4,4 mil receitas ativas:
- Arvore: ~40 mil nós;
- ReceitaInsumo: ~26 mil linhas;
- LigacaoSetor: ~4 mil linhas;
- profundidade máxima: 5 níveis de receita;
- carga: menos de um minuto.

### Regras de cálculo

- **Só receitas:** a árvore e a lista de materiais ignoram rendimentos (`INDRECEITARENDTO = 'C'` na CTE `cad`). Um produto feito só por rendimento, como um corte de desossa, aparece como insumo folha.
- **Produtor principal:** quando um produto tem mais de um produtor, vale a receita antes do rendimento e, entre receitas, o menor código. A escolha é feita por receita pai, com `ROW_NUMBER` em `filhos_prod`.
- **Fator acumulado:** fator do pai × `QTDUNIDUTILIZADA` × peso ÷ `QTDUNIDPRODUZIDA`. O peso rateia por quantidade entre os produtos finais do cadastro. O cálculo supõe unidades coerentes entre os níveis.
- **Lista de materiais:** soma do fator nos nós com `[Folha Explosão] = 'Sim'`, ou seja, ramo principal e sem produtor.
- **Final ou Intermediário:** a receita é Intermediário se o seu produto é componente de alguma outra receita ativa.
- **Setor do nó (`[Origem Setor]`):** setor da receita que produz o nó. Entra como atributo, por mapeamento, para não criar laço com Receitas.
- **Nó Folha:** "Sim" quando o `%NoID` não é pai de nenhum outro nó. É usado para colorir o organograma.
- **Ligações entre setores:** `HierarchyBelongsTo` sobre a Arvore. Cada nó-receita é fornecedor de todas as receitas acima dele no caminho. Pares repetidos ficam com a menor distância.
- **Ciclos:** a CTE usa `CYCLE seqcad_origem`. Se o cadastro tiver ciclos, os caminhos cíclicos são descartados.

## 5. Variáveis

| Variável | Uso |
|---|---|
| `vQtdProduzir` | Quantidade a produzir, na pasta Insumos da receita. Criada no script com valor 1 se não existir |
| `vQtdInsumo` | Quantidade disponível do insumo, na pasta Produção possível |
| `vCodReceita`, `vCodProduto` | Caixas de código de receita e de produto final, nas pastas Árvore e Insumos da receita. Compartilhadas entre as pastas |
| `vCodInsumo` | Caixa de código de insumo, nas pastas Uso do insumo e Produção possível |

## 6. Pastas

Todas as pastas têm, no topo, uma faixa centralizada com o botão-link "Criado com Claude. GitHub ↗", que abre este repositório. Abaixo dela fica a barra de navegação com seis botões, nesta ordem:
**Lista de receitas · Árvore · Dependências setores · Insumos da receita · Uso do insumo · Produção possível**.
O botão da pasta atual aparece mais escuro. A grade das pastas tem 84 × 43 células.

Os IDs das pastas são mantidos quando o `.qvf` é importado, e os botões de navegação dependem deles.

| Pasta (nome no app) | ID | Rótulo do botão |
|---|---|---|
| Lista de receitas | `696a2nlvwd` | Lista de receitas |
| Árvore de dependências | `m12vcjrpgv` | Árvore |
| Dependências entre setores | `br636uqshz` | Dependências setores |
| Insumos folha de receita | `aba51ahugn` | Insumos da receita |
| Onde o insumo é usado | `nydiqj98f2` | Uso do insumo |
| Produção possível com o insumo | `f576sgd2i0` | Produção possível |

### 6.1 Lista de receitas
- Tabela com as colunas Cód. receita, Receita, Cód. produto final, Produto final, Subreceitas distintas, Subreceitas (com repetição) e Caminhos.
  - Subreceitas distintas: `Count(DISTINCT {<[Tipo Nó]={'Receita'}, [Nível]={">1"}>} [Origem Código])`
  - Subreceitas com repetição: `Count({<[Tipo Nó]={'Receita'}, [Nível]={">1"}>} %NoID)`
  - Caminhos: `Count({<[Nó Folha]={'Sim'}>} %NoID)`
- As larguras das colunas são fixas em pixels.
- Filtros: Setor, Tipo, Profundidade.

### 6.2 Árvore de dependências
- Organograma com as dimensões `%NoID` e `%NoPaiID`.
  - Condição de cálculo: `Count(DISTINCT %SeqReceitaRaiz) = 1`. Sem ela, o objeto estoura o limite de dados.
  - Título: `Receita <código> - <nome>  →  Produz <código produto> - <produto>`. Sem uma receita selecionada, o título fica "Árvore de dependências".
- Cartão de cada nó:
  - Título: `Only([Nó])`, com `(código do produto)` no fim quando o nó é uma receita.
  - Subtítulo: `Only([Tipo Nó]) & ': ' & Only([Origem Cadastro]) & ' (código da receita)'`.
  - Linha de baixo: `<setor> | Usa: <Qtd Utilizada> | Produz: <Qtd Produzida>`, ou `Produto final` na raiz.
  - Fundo: `If(Only([Nó Folha]) = 'Sim', RGB(220,245,220), RGB(255,255,255))`.
- Caixas **Cód. receita** e **Cód. produto final**, cada uma com o botão Filtrar (ver seção 7).
- Filtros: Produto da receita, Receita, Receita Setor.

### 6.3 Dependências entre setores
- **Sankey:** Setor Fornecedor → Setor Consumidor, medida `Count(%LigacaoID)`. Inclui as ligações dentro do mesmo setor.
- **Tabela:** Setor fornecedor, Cód. fornecedora, Receita fornecedora, Produto fornecido, Setor consumidor, Cód. consumidora, Receita consumidora, Níveis entre elas (`Min([Lig Distância])`) e Qtd utilizada (só nas diretas).
- Dois painéis de filtro: Setor fornecedor com Mesmo setor / Entre setores, e Setor consumidor.

### 6.4 Insumos folha de receita
- Entrada "Quantidade a produzir" (`vQtdProduzir`) e caixas de código de receita e de produto.
- **Tabela do produto final:** Código produto, Produto final, Setor e Qtd produzida (`Only({<[Nível]={1}>} [Qtd Produzida])`). Título: "Produto final da receita <código> - <nome>".
- **Tabela de insumos**, com o título "Insumos folha necessários para produzir o produto <código> - <descrição>". Colunas:
  - Qtd por 1 un.: `Sum([Qtd Insumo por Unidade])`;
  - Qtd por 1 receita: `Sum([Qtd Insumo por Unidade]) * Only(TOTAL {<[Nível]={1}>} [Qtd Produzida])`;
  - Qtd para N un., com o valor de `vQtdProduzir`;
  - % do total: `Sum(x) / Sum({<[Insumo]=,[Insumo Código]=>} TOTAL x)`;
  - Nº de caminhos.

  A condição de cálculo é uma receita selecionada.
- Filtros: Produto da receita, Receita.

### 6.5 Onde o insumo é usado
- Caixa **Cód. insumo**.
- Tabela com as receitas que usam o insumo, com setor, receita, produto, quantidade por unidade, quantidade para N unidades e número de caminhos. Não tem linha de totais. Condição de cálculo: `Count(DISTINCT %SeqInsumo) = 1`.
- Filtros: Insumo (folha), Setor, Tipo.

### 6.6 Produção possível com o insumo
- Entrada "Quantidade disponível do insumo" (`vQtdInsumo`) e caixa **Cód. insumo**.
- Tabela por receita:
  - Insumo por 1 un. do produto;
  - Produto final possível: `$(vQtdInsumo) / Sum([Qtd Insumo por Unidade])`;
  - Unidades inteiras: `Floor(Round(x, 0.001))`. O arredondamento evita que 239,9999 vire 239.
- Cada linha supõe que todo o insumo vai para aquela receita: as linhas são alternativas e não se somam. O cálculo também supõe que os outros insumos estão disponíveis.

## 7. Caixas de código e botão Filtrar

- Cada caixa é uma "Entrada de variável" com um botão **Filtrar** ao lado.
- Ações do botão de receita:
  1. `clearAll`;
  2. `selectMatchingValues` no campo `[Receita Código]`, com a busca `=[Receita Código] = If(Len(Trim(vCodReceita)) > 0, Trim(vCodReceita), Only({1<[Receita Produto Código]={"$(=Trim(vCodProduto))"}>} [Receita Código]))`;
  3. `setVariable vCodProduto = '=Only([Receita Produto Código])'`, que preenche a outra caixa.
- O botão de produto é simétrico. O botão de insumo faz `clearAll` e `selectMatchingValues` em `[Insumo Código]`.
- **Por que a busca é uma expressão `=...`:** o valor de uma ação comum do botão é calculado **antes** de a caixa repassar o texto digitado. O primeiro clique aplicava o código antigo. A busca é avaliada pelo motor do Qlik na hora da seleção, já com o valor novo.
- **Por que há uma alternativa com a outra variável:** depois do `clearAll`, a caixa preenchida automaticamente, que é uma fórmula ligada à seleção, fica vazia. O botão então usa o código da outra caixa.
- **Limitações:** não é possível filtrar com Enter, porque a caixa de variável não dispara ações. Clicar duas vezes seguidas em "Filtrar" numa caixa preenchida automaticamente, sem digitar nada, limpa a seleção.

## 8. Como validar depois de carregar

- **Ciclos:** o número de receitas com árvore deve ser igual ao de receitas ativas com produto final ativo. Se a CTE encontrar ciclos, a carga continua, mas os caminhos cíclicos somem.
- **Nós duplicados:** `Count(%NoID) = Count(DISTINCT %NoID)` na Arvore.
- **Chaves sintéticas:** o visualizador do modelo de dados não deve mostrar nenhuma.
- **Lista de materiais:** escolha 2 ou 3 receitas conhecidas e compare a soma da "Qtd por 1 receita" com o que a produção usa de fato. Diferenças grandes costumam indicar unidades incoerentes entre os níveis.
- **Botões de navegação e link:** depois de importar, clique em cada botão para confirmar que leva à pasta certa.

## 9. Decisões e aprendizados técnicos

- **Explosão:** é feita **no Oracle**, com CTE recursiva e `CYCLE`, e não no Qlik. `CAST(... AS VARCHAR2(4000))` no membro âncora evita erro de tamanho no caminho.
- **Comentários `--` no SQL:** quebram o conector com o erro "InvalidArgument". Use `//` só fora do bloco SQL.
- **`NOCONCATENATE`:** uma tabela temporária com os mesmos campos de outra é concatenada automaticamente; use `NOCONCATENATE`.
- **Setor como mapeamento:** o setor entra por mapeamento, e não como tabela, para não criar um laço Receitas–Arvore.
- **Filtros só do objeto principal:** campos da `Arvore` não vão para os painéis de filtro, porque selecioná-los corta o organograma. Cada painel tem só os campos que mudam o objeto principal da pasta.
- **Sankey (`qlik-sankey-chart-ext`):** só funciona quando criado pela interface, arrastando do painel de ativos. Criado por API, fica com o tipo errado ou vazio. Depois de criado, dá para configurar por API.
- **Rótulos dos botões:** o botão de ação não quebra linha no rótulo, porque a fonte diminui até caber. Por isso os rótulos são curtos. `style.font.size` é um limite máximo relativo à altura.
- **Objeto Texto e imagem:** não aceitou a marcação de tamanho de fonte e quebrava o link em duas linhas. O crédito virou um botão com a ação `openWebsite`, fundo branco e texto sublinhado.
- **Tabela simples (`table`):** as larguras ficam em `qHyperCubeDef.columnWidths`, em pixels. `qDef.columnWidth` em porcentagem não teve efeito.
- **Formato numérico:** o formato `'#,##0.###'` mostrou zeros extras. Nos cartões do organograma, os números aparecem como estão no cadastro.
- **Ordem dos filtros num painel:** segue a ordem de criação. Para reordenar, os filtros precisam ser recriados.
- **Edições por API:** foram feitas pela Engine API, por WebSocket (`wss://<tenant>/app/<id>?qlik-csrf-token=...`), com `SetScript`, `DoReload`, `SetProperties`, `SetFullPropertyTree`, `CreateVariableEx` e `DoSave`.

## 10. Verificações de cadastro recomendadas

Problemas que o app ajuda a encontrar e que, na prática, costumam aparecer:
- **Receitas sem setor de produção:** atrapalham a leitura da pasta Dependências setores.
- **Kits que usam outro kit** como componente: pode ser intencional ou duplicidade.
- **Produto final que não combina com o nome da receita:** em geral, erro de digitação do código.
- **Receitas ativas com "INATIVO", "REUTILIZAR" ou "TESTE" no nome:** o campo `[Receita Cadastro Suspeito]` as marca.
- **Receitas sem componentes ativos:** aparecem como folha (verde) no organograma.
- **Receitas ativas sem produto final ativo:** não aparecem na árvore.

## 11. Pendências e ideias

- **% do total:** decidir se água e itens sem custo entram, porque costumam pesar muito.
- **Unidade de medida do insumo:** identificar a tabela no Consinco e mostrar a unidade.
- **Custo por insumo:** preço médio × quantidade.
- **Consumo real por período:** com os movimentos de produção ou as vendas.
- **Gráfico de rede "onde é usado":** a partir de `RedeArestas`.
- **Navegação:** avaliar uma pasta de entrada (menu) no lugar da barra de botões.
