1. Задеплоить StatfulSet с Prometheus
2. Задеплоить DaemonSet с node_exporter
3. Настроить Prometheus так чтобы он собирал метрики с node_exporter
4. Задеплоить Deployment с Grafana
5. В Grafana подключить Prometheus как Datasource по имени Service
6. В Grafana добавить дашборд для node_exporter
7. В Grafana настроить алерт с правилом: Если использование RAM больше 10% отправлять оповещение
