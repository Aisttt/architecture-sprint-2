MongoDB Sharding Project
Этот проект настраивает MongoDB с шардированием для повышения производительности. В конфигурации используется два шарда, сервер конфигурации и роутер (mongos). Основная база данных называется somedb, а коллекция — helloDoc.

Шаги для запуска проекта
Запустите проект с помощью Docker Compose:

bash
Копировать код
docker compose up -d
Убедитесь, что все контейнеры запущены:

bash
Копировать код
docker ps
Инициализация шардирования
Следуйте этим шагам для настройки шардирования и инициализации коллекции.

1. Инициализация набора реплик для сервера конфигурации
Запустите команду для инициализации сервера конфигурации:

bash
Копировать код
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF
rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [{ _id: 0, host: "configSrv:27017" }]
})
EOF
2. Инициализация наборов реплик для шардов
Запустите команду для инициализации первого шарда:

bash
Копировать код
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: "shard1",
  members: [{ _id: 0, host: "shard1:27018" }]
})
EOF
Запустите команду для инициализации второго шарда:

bash
Копировать код
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
rs.initiate({
  _id: "shard2",
  members: [{ _id: 0, host: "shard2:27019" }]
})
EOF
3. Настройка роутера для добавления шардов
Добавьте шард shard1 и shard2 в систему шардирования через роутер mongos:

bash
Копировать код
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1:27018")
sh.addShard("shard2/shard2:27019")
EOF
4. Включение шардирования для базы данных somedb и коллекции helloDoc
Включите шардирование для базы данных somedb:

bash
Копировать код
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.enableSharding("somedb")
EOF
Создайте коллекцию helloDoc с индексом, необходимым для шардирования:

bash
Копировать код
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb
db.createCollection("helloDoc")
db.helloDoc.createIndex({ _id: "hashed" })
sh.shardCollection("somedb.helloDoc", { _id: "hashed" })
EOF
Проверка количества документов
После настройки шардирования можно проверить количество документов в коллекции helloDoc в каждом шарде. Добавьте не менее 1000 документов в коллекцию для тестирования (пример с использованием скрипта ниже).

Добавление документов в коллекцию helloDoc
Используйте следующую команду, чтобы добавить 1000 документов в коллекцию helloDoc:

bash
Копировать код
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb
for (let i = 0; i < 1000; i++) {
  db.helloDoc.insert({ _id: i, message: "Hello from document " + i })
}
EOF
Отображение общего количества документов в коллекции helloDoc
Проверьте общее количество документов в коллекции:

bash
Копировать код
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
Отображение количества документов в каждом шарде
Проверьте количество документов в каждом шардовом инстансе:

bash
Копировать код
# Количество документов в shard1
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF

# Количество документов в shard2
docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
Успешное выполнение
После выполнения всех шагов вы должны увидеть общее количество документов (≥ 1000) и распределение документов по шардовым инстансам shard1 и shard2.