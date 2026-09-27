# Desafio de Projeto — Star Schema com Power BI (Financial Sample)

## 📌 Objetivo

Este projeto foi desenvolvido como desafio da Formação Power BI Analyst (DIO). O objetivo era partir de uma única tabela — a *Financial Sample* — e, a partir dela, derivar um modelo dimensional em **Star Schema**, com uma tabela fato central e suas tabelas de dimensão.

## 🗂️ Estrutura do modelo

A partir da tabela original (`financials`), mantida oculta no relatório como `financials Origem` (backup), foram criadas as seguintes tabelas:

| Tabela | Tipo | Grão (nível de detalhe) | Descrição |
|---|---|---|---|
| `F_Vendas` | Fato | 1 linha por transação | Tabela central, com `SK_ID` (chave substituta), `ID_Produto`, `ID_Categoria`, Date, Discount Band, Sale Price, Segment, Units Sold |
| `D_Produtos` | Dimensão | 1 linha por produto | Produto, com métricas agregadas (contagem de unidades vendidas, valor mínimo/máximo/médio/mediano de venda, média de manufatura) |
| `D_Categoria` | Dimensão | 1 linha por combinação Segmento + País | `ID_Categoria`, Segment, Country |
| `D_Calendario` | Dimensão | 1 linha por data | Gerada via DAX, com colunas Ano, Mês e NúmeroMês |
| `D_Descontos` | Dimensão (satélite da fato) | 1 linha por transação | Discount Band, Discounts, ligada por `SK_ID` |
| `D_Produtos_Detalhes` | Dimensão (satélite da fato) | 1 linha por transação | Discount Band, Manufacturing Price, Sale Price, ligada por `SK_ID` |
| `D_Detalhes` | Dimensão (satélite da fato) | 1 linha por transação | COGS, Gross Sales, Profit, ligada por `SK_ID` |

## 🔗 Relacionamentos

Todas as ligações saem de `F_Vendas` (modelo em estrela, sem relação entre dimensões):

| De (F_Vendas) | Para | Cardinalidade |
|---|---|---|
| `ID_Produto` | `D_Produtos[ID_Produto]` | Muitos para um (*:1) |
| `ID_Categoria` | `D_Categoria[ID_Categoria]` | Muitos para um (*:1) |
| `Date` | `D_Calendario[Date]` | Muitos para um (*:1) |
| `SK_ID` | `D_Descontos[SK_ID]` | Um para um (1:1) |
| `SK_ID` | `D_Produtos_Detalhes[SK_ID]` | Um para um (1:1) |
| `SK_ID` | `D_Detalhes[SK_ID]` | Um para um (1:1) |

### Por que `SK_ID` em vez de `ID_Produto` em algumas dimensões?

`D_Descontos`, `D_Produtos_Detalhes` e `D_Detalhes` foram construídas mantendo o grão de transação (uma linha por venda), não o grão de produto. Como `ID_Produto` se repete várias vezes nessas tabelas (várias transações do mesmo produto), ele não é uma chave única e não permite relacionamento direto de `F_Vendas` para elas.

A solução foi adicionar uma **coluna de índice sequencial (`SK_ID`)** em cada uma dessas tabelas, correspondente linha a linha ao `SK_ID` já existente em `F_Vendas`. Isso garante uma chave única em ambos os lados, permitindo o relacionamento **um para um**.

### Como a `ID_Categoria` foi trazida para `F_Vendas`

Como nem `Segment` nem `Country` são únicos isoladamente em `D_Categoria` (25 combinações possíveis, mas poucos valores distintos em cada coluna), relacionar diretamente por esses campos resultaria em cardinalidade muitos-para-muitos.

A solução foi usar **Mesclar Consultas (Merge)** no Power Query: `F_Vendas` foi mesclada com `D_Categoria` pela combinação de `Segment` + `Country`, trazendo o campo `ID_Categoria` já correspondente para dentro da fato. A partir daí, o relacionamento passou a ser feito por essa chave única.

## 🧮 Funções DAX utilizadas

**Criação da tabela de calendário:**
```dax
D_Calendario = CALENDAR(MIN(F_Vendas[Date]), MAX(F_Vendas[Date]))
```

**Colunas auxiliares em `D_Calendario`:**
```dax
Ano = YEAR(D_Calendario[Date])
Mês = FORMAT(D_Calendario[Date], "MMMM")
NúmeroMês = MONTH(D_Calendario[Date])
```

## ⚙️ Processo de construção (Power Query)

1. Importação da tabela `financials` (Financial Sample)
2. Duplicação da tabela original em cada consulta de dimensão, seguida das transformações específicas:
   - **D_Produtos**: Agrupar por (Agregar por) produto, com soma/mín/máx/média/mediana das métricas de venda, mais coluna de índice
   - **D_Descontos**: seleção de colunas de desconto, coluna condicional `ID_Produto`, remoção de duplicatas, coluna de índice `SK_ID`
   - **D_Categoria**: seleção de Segment/Country, remoção de duplicatas, coluna de índice `ID_Categoria`
   - **D_Produtos_Detalhes** e **D_Detalhes**: seleção das colunas específicas, coluna condicional `ID_Produto`, coluna de índice `SK_ID`
   - **F_Vendas**: seleção das colunas de fato, coluna condicional `ID_Produto`, coluna de índice `SK_ID`, mesclagem com `D_Categoria` para trazer `ID_Categoria`
3. Ocultação da tabela `financials Origem` no modo de relatório (mantida apenas como backup)
4. Criação da `D_Calendario` via DAX
5. Construção dos relacionamentos na view de Modelagem, validando a cardinalidade de cada um

## 📁 Arquivos deste repositório

- `Projeto_DIO_Modulo4.pbix` — arquivo do Power BI com o modelo completo
- `esquema_estrela.png` — print da view de Modelagem, mostrando o diagrama em estrela finalizado
- `README.md` — este documento

## 👤 Autor

Matheus — desafio da Formação Power BI Analyst, DIO.
