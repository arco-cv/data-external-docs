# data-external-docs

<!-- agent-readiness-badge-begin -->
[![Agent Readiness: ❌ Sem Nota](https://img.shields.io/badge/Agent_Readiness-%E2%9D%8C_Sem_Nota-red)](.claude/docs/agent-readiness-report.md)
<!-- agent-readiness-badge-end -->

Ferramenta de documentação automática de colunas do BigQuery.

Roda como pod efêmero disparado pelo Airflow: lê o schema real de um dataset,
baixa a documentação externa da API de origem, gera descrições em PT-BR com LLM,
reconcilia no `schema.yml` versionado do `arco-cv/dbt` e abre um PR — sem publicar nada.

> Scaffold em construção (DP-7180). Os módulos funcionais chegam nas issues seguintes.
