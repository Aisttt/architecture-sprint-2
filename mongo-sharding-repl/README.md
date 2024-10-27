# MongoDB Sharding and Replication Project

Этот проект настраивает MongoDB с шардированием и репликацией для повышения отказоустойчивости. Конфигурация включает два шарда, каждый с набором из трех реплик (Primary и два Secondary), сервер конфигурации и роутер `mongos`. Основная база данных называется `somedb`, а коллекция — `helloDoc`.

## Шаги для запуска проекта

1. Запустите проект с помощью Docker Compose:
   ```bash
   docker compose up -d
   ```
   
   Если требуется удалить контейнеры 
   ```bash
   docker rm -f $(docker ps -aq)
   ```

2. Убедитесь, что все контейнеры запущены:
   ```bash
   docker ps
   ```

## Настройка репликации

### 1. Инициализация набора реплик для сервера конфигурации

```bash
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF
rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [{ _id: 0, host: "configSrv:27017" }]
})
EOF
```

### 2. Инициализация наборов реплик для шардов

**Инициализация первого шарда:**
```bash
docker compose exec -T shard1-primary mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host: "shard1-primary:27018" },
    { _id: 1, host: "shard1-secondary1:27019" },
    { _id: 2, host: "shard1-secondary2:27020" }
  ]
})
EOF
```

**Инициализация второго шарда:**
```bash
docker compose exec -T shard2-primary mongosh --port 27021 --quiet <<EOF
rs.initiate({
  _id: "shard2",
  members: [
    { _id: 0, host: "shard2-primary:27021" },
    { _id: 1, host: "shard2-secondary1:27022" },
    { _id: 2, host: "shard2-secondary2:27023" }
  ]
})
EOF
```

### 3. Настройка роутера для добавления шардов

```bash
docker compose exec -T mongos_router mongosh --port 27024 --quiet <<EOF
sh.addShard("shard1/shard1-primary:27018")
sh.addShard("shard2/shard2-primary:27021")
EOF
```

### 4. Включение шардирования для базы данных `somedb` и коллекции `helloDoc`

Включите шардирование для базы данных `somedb`:
```bash
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.enableSharding("somedb")
EOF
```

Создайте коллекцию `helloDoc` с индексом, необходимым для шардирования:
```bash
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb
db.createCollection("helloDoc")
db.helloDoc.createIndex({ _id: "hashed" })
sh.shardCollection("somedb.helloDoc", { _id: "hashed" })
EOF
```

## Проверка работы шардирования и репликации

Добавьте 1000 документов в коллекцию `helloDoc`:
```bash
docker compose exec -T mongos_router mongosh --port 27024 --quiet <<EOF
use somedb
for (let i = 0; i < 1000; i++) {
  db.helloDoc.insert({ _id: i, message: "Hello from document " + i })
}
EOF
```

Проверьте общее количество документов:
```bash
docker compose exec -T mongos_router mongosh --port 27024 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Проверьте количество документов в каждом шарде:
```bash
docker compose exec -T shard1-primary mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF

docker compose exec -T shard2-primary mongosh --port 27021 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```
