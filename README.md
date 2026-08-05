# data-external-docs

Ferramenta de documentação automática de colunas do BigQuery.

Roda como pod efêmero disparado pelo Airflow: lê o schema real de um dataset,
baixa a documentação externa da API de origem, gera descrições em PT-BR com LLM,
reconcilia no `schema.yml` versionado do `arco-cv/dbt` e abre um PR — sem publicar nada.

> Scaffold em construção (DP-7180). Os módulos funcionais chegam nas issues seguintes.
