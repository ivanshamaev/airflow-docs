# Airflow для MLOps

**MLOps** охватывает всё необходимое для эксплуатации ML-моделей в продакшене. Airflow — инструмент-агностик оркестрации: он может управлять шагами в любом MLOps-инструменте с API. Плюсы: использование существующей экспертизы по Airflow, общая платформа для data/ML инженеров, [интеграции](https://registry.astronomer.io/providers), retries и branching, идемпотентные пайплайны и backfill, выбор compute под задачу (GPU, Spark), мониторинг и алерты, [кастомные модули](../astronomer-dags/custom-hooks-operators.md) и [плагины](airflow-plugins.md).

**LLMOps** (Large Language Model Operations): fine-tuning, RAG, prompt engineering. Airflow подходит для оркестрации пайплайнов инференса и GenAI.

Компоненты MLOps: **BusinessOps** (governance, стратегия), **DevOps** (CI/CD, версионирование, IaC), **DataOps** (хранение данных, препроцессинг, [data quality](https://www.astronomer.io/docs/learn/data-quality)), **ModelOps** (обучение, деплой, мониторинг моделей). Для обучения и инференса важно правильно выбрать compute (KubernetesPodOperator, отдельные очереди воркеров). Интеграции: MLFlow, Weights & Biases, Great Expectations, Open Lineage и др.

Дополнительно: [GenAI и MLOps](https://www.astronomer.io/docs/learn/category/mlops), [Ask Astro](https://github.com/astronomer/ask-astro), [Astronomer Academy — GenAI](https://academy.astronomer.io/introduction-to-genai-with-apache-airflow).

---

[← Cluster policies](advanced-cluster-policies.md) | [К содержанию](README.md) | [Плагины →](airflow-plugins.md)
