# Dashboard executivo sobre o e-commerce da Olist

Modelagem de dados em escala (processamento em Parquet + DuckDB) e um dashboard executivo em Power BI construídos sobre +100 mil pedidos da Olist, a maior plataforma de marketplace do Brasil, para responder uma pergunta central: **onde o negócio está crescendo, onde está estagnado e o que fazer a respeito.**

O projeto cobre o ciclo completo de um trabalho de BI: da ingestão de dados brutos e modelagem dimensional em SQL, passando pela construção de um dashboard executivo em Power BI (com DAX), até a análise de negócio e recomendações estratégicas.

## O que esse projeto responde

Um marketplace com dados brutos, imprecisos e mal documentados só vira ferramenta de decisão depois de passar por modelagem, tratamento e camadas de agregação corretas. Este projeto usa a base pública da Olist (2016–2018) para construir esse caminho do zero — e responder três perguntas de negócio centrais: **como o desempenho da plataforma evoluiu ao longo do tempo, onde estão as maiores concentrações de risco e oportunidade geográfica, e o que explica a insatisfação de parte dos clientes.**

## Principais achados

- **Crescimento acelerado, seguido de platô.** O faturamento saiu de ~R$ 200 mil/mês para mais de R$ 1 milhão/mês em um ano, com picos evidentes no Dia das Mães (mai/2017 e mai/2018) e no maior faturamento histórico da plataforma, na Black Friday de nov/2017. A partir de maio de 2018, o crescimento entra em um platô que merece investigação.
- **Concentração geográfica extrema.** A região Sudeste responde por 68 mil pedidos contra 14 mil da região Sul (2ª colocada) e fatura sozinha R$ 10,62 milhões — quase o dobro do restante do país somado (R$ 5,83 milhões). A base de vendedores segue o mesmo padrão (2,3 mil no Sudeste vs. 0,7 mil no Sul).
- **Atraso na entrega é o principal preditor de insatisfação.** Apenas 7% dos pedidos atrasam, mas esses pedidos levam em média 33 dias para chegar (contra 12 dias da média geral) e recebem nota média de 2,27, muito abaixo dos 4,09 da média geral. O atraso não está concentrado em nenhuma região específica — é proporcional ao volume de pedidos de cada uma.
- **O tíquete médio é puxado por categorias específicas.** Beleza & saúde, relógios & presentes e cama, mesa & banho lideram o faturamento e sustentam o tíquete médio mais alto — um padrão consistente com hipótese de "compra para presentear", reforçada pela concentração em datas comemorativas.

## Como cheguei lá

1. **Ingestão e processamento:** os arquivos `.csv` brutos da Olist foram convertidos para `.parquet` via Pandas, o que resolveu um gargalo real de performance na etapa de carga — permitindo carregar todas as tabelas em um banco DuckDB em menos de 2 segundos.
2. **Modelagem dimensional em SQL:** usando DDL e DML no DuckDB, as tabelas brutas foram reorganizadas em um esquema de **constelação de fatos** (não uma estrela simples, já que existem múltiplos eventos de negócio — pedidos, pagamentos e avaliações — que não compartilham a mesma granularidade). Tabelas redundantes (ex: traduções de categoria já presentes em outra tabela) e de granularidade excessiva (geolocalização exata) foram deixadas fora do modelo semântico, mas preservadas na camada intermediária.
3. **Checagem de integridade e chaves:** cada tabela de dimensão foi validada comparando contagem total de linhas com contagem de valores únicos de chave, o que revelou, por exemplo, que `id_cliente` não identifica um cliente único — `id_unica_cliente` é a chave correta para análises de cliente, ainda que `id_cliente` precise ser mantida como chave estrangeira de outras tabelas.
4. **Feature engineering:** criação das colunas `regiao` (agrupando estados por macrorregião do IBGE), `dias_totais_ate_entrega`, `status_pedidos_entregues` (adiantado/atrasado) e `adiantos_e_atrasos`, todas usadas diretamente nas análises de eficiência operacional do dashboard.
5. **Modelagem no Power BI e DAX:** o modelo semântico final ficou estruturado em 3 dimensões (`dim_clientes`, `dim_produtos`, `dim_vendedores`) e 4 fatos (`f_pedidos`, `f_itens_pedido`, `f_pagamentos`, `f_avaliacao_pedidos`), com medidas DAX para as métricas centrais (faturamento, receita líquida de frete, tíquete médio, tempo médio de entrega, taxa de entrega no prazo).
6. **Dashboard executivo em duas páginas:** "Visão Executiva" (evolução temporal, distribuição regional, faturamento por categoria) e "Satisfação & Entrega" (correlação entre nota de avaliação, prazo de entrega e região), com navegação entre páginas via botões customizados.

## Desafios técnicos enfrentados

Vale destacar dois problemas reais encontrados na modelagem, porque o processo de diagnóstico diz mais sobre a análise do que o resultado final:

- **Bug de propagação de filtro entre fatos.** Como `f_pagamentos` e `f_itens_pedido` se conectam através de `f_pedidos` (constelação de fatos, não estrela simples), o filtro de categoria de produto não propagava corretamente até a medida de faturamento — todas as categorias retornavam o mesmo total geral. A correção foi feita na própria medida DAX, usando `TREATAS` para forçar a ponte de filtro entre as tabelas fato sem alterar a direção de cross-filtering do modelo (o que poderia gerar ambiguidade em outros visuais).
- **Integridade de rótulos de KPI.** Um KPI inicialmente batizado de "Lucro" na verdade representava apenas faturamento menos frete — sem descontar repasse a vendedores ou custos operacionais, que não estão disponíveis nesta base. Foi renomeado para "Receita Líquida de Frete" para não sugerir uma leitura de lucratividade que os dados não sustentam.

## Resultados principais

| Métrica | Valor |
| --- | --- |
| Faturamento total (2016–2018) | R$ 16,45 Mi |
| Receita líquida de frete | R$ 14,20 Mi |
| Quantidade de pedidos | 99 Mil |
| Tíquete médio | R$ 165,47 |
| Taxa de entrega no prazo | 93% |
| Tempo médio de entrega (geral / atrasados) | 12 dias / 33 dias |
| Nota média de avaliação (geral / pedidos atrasados) | 4,09 / 2,27 |

## Reflexão estratégica

Com base nos achados, três oportunidades de crescimento se destacam: **expansão regional** (o Sudeste concentra a operação de forma desproporcional, e há espaço evidente para captar vendedores e clientes em outras regiões), **preparação para sazonalidade** (os picos de Dia das Mães e Black Friday indicam onde concentrar esforço de marketing e logística) e **definição de estratégia entre "compra como hábito" e "compra como presente"**, a depender de qual comportamento a Olist queira reforçar nas categorias líderes de faturamento.

Se contratada para atuar nesse contexto, as duas primeiras ações seriam: (1) uma análise de série temporal a partir de 2019, ano de entrada da Shopee no mercado brasileiro, para entender o impacto competitivo sobre o crescimento observado até 2018; e (2) uma rodada de alinhamento com áreas de negócio para auditar a confiabilidade de chaves de identificação (como `id_cliente` vs. `id_unica_cliente`) antes de expandir o modelo analítico.

## Limitações

Os dados públicos da Olist não incluem custos, taxas ou margem operacional da própria plataforma — apenas o valor transacionado entre vendedores e clientes. Isso significa que qualquer leitura de "saúde financeira" deste projeto se refere ao volume de negócios movimentado pelos e-commerces que usam a Olist, não à lucratividade da Olist como empresa. Uma leitura completa da sustentabilidade do negócio exigiria dados adicionais de custo e repasse.

## Stack técnica

`Python` `Pandas` `DuckDB` `SQL` `Power BI` `DAX`

## Estrutura do projeto

```
├── raw/                      # Dados brutos (.csv) — não versionados por tamanho
├── processed/                # Dados tratados em formato Parquet
├── sql/                      # Scripts DDL/DML de modelagem (schema de constelação de fatos)
├── notebooks/
│   └── 1_Processamento.ipynb # Processamento e conversão para Parquet
├── dashboard/
│   └── olist-dashboard.pbix  # Arquivo Power BI
├── docs/
│   └── relatorio-executivo-olist.pdf  # Relatório gerencial completo (processo, insights, recomendações)
├── requirements.txt
└── README.md
```

## Como rodar

```bash
git clone https://github.com/ClaudiaSobral/olist-bi.git
cd olist-bi
pip install -r requirements.txt
```

Os dados brutos da Olist podem ser baixados no [Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) e devem ser colocados em `raw/`. Rode `notebooks/1_Processamento.ipynb` para gerar os arquivos Parquet, aplique os scripts de `sql/` no DuckDB para reconstruir o modelo semântico e abra `dashboard/olist-dashboard.pbix` no Power BI Desktop para explorar o dashboard interativo.

## Aprofundamento

O relatório gerencial completo — com o detalhamento do processo de modelagem, decisões de tratamento de dados, análise de negócio completa e reflexão estratégica — está disponível em [`docs/relatorio-executivo-olist.pdf`](./docs/relatorio-executivo-olist.pdf).

## Sobre mim

Sou a Claudia Sobral, ex-animadora de TV migrando para Dados — trago dessa trajetória a capacidade de transformar números em histórias claras. [claudiasobral.com](https://claudiasobral.com/) · [LinkedIn](https://www.linkedin.com/in/claudia-sobral/)
