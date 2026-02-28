# Мультиязычность (Multilanguage)

В Airflow 3 можно писать **SDK для других языков**, чтобы определять задачи не только на Python. Экспериментальный **Go SDK** поставляется с Airflow 3.0; статус и код: [go-sdk](https://github.com/apache/airflow/tree/main/go-sdk). Это снижает привязку к одному языку и упрощает перенос существующих воркфлоу с других платформ.

Альтернативы без SDK: **KubernetesPodOperator** — запуск любого Docker-образа с кодом на любом языке (см. [пример в документации](https://www.astronomer.io/docs/learn/kubepod-operator#example-use-the-kubernetespodoperator-to-run-a-script-in-another-language)); **BashOperator** — запуск скрипта (например, R или JavaScript) через интерпретатор в окружении воркера. Для изоляции и воспроизводимости предпочтительнее образ в Kubernetes.

Подробнее: [Multilanguage](https://www.astronomer.io/docs/learn/airflow-multilanguage), [BashOperator — другой язык](https://www.astronomer.io/docs/learn/bashoperator#example-run-a-script-in-another-programming-language).

---

[← Логирование](logging.md) | [К содержанию](README.md) | [Динамические DAG →](dynamic-dags.md)
