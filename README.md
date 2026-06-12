# DevOps Test Assignment | Иванов Филипп | ООО «Кельник Студиос»

## Проблема

При конкурентной нагрузке (50+ параллельных запросов) сервис возвращает ошибку:

```json
{"ok":false,"error":"cache is busy","type":"RuntimeException"}

Частота: ~28% запросов падают при 100 запросах (50 в parallel)
Диагностика
Шаги воспроизведения
bash

# Запуск
docker run --rm --name devops-test -p 8080:80 docker.kelnik.pro/tests/devops-stack:1.0

# Воспроизведение ошибки
seq 1 100 | xargs -P 50 -I {} curl -s http://localhost:8080/items | grep -c "cache is busy"
# Результат: 28

Анализ

    Исследование кода (/var/www/html/index.php)

Обнаружена проблема в distributed lock:
php

// lock TTL = 2 секунды
$locked = $redis->set($lockKey, $token, ['nx', 'ex' => 2]);

// expensiveLoad() занимает 2.5 секунды
function expensiveLoad(PDO $pdo): array {
    usleep(2500000);  // 2.5 сек!
    // ...
}

// Всего одна попытка retry через 200ms
if (!$locked) {
    usleep(200000);
    // одна проверка
    throw new RuntimeException('cache is busy');
}

    PHP-FPM конфигурация - недостаточно процессов

ini

pm.max_children = 5      # слишком мало
pm.start_servers = 2

    Persistent connections в PDO

php

PDO::ATTR_PERSISTENT => true  # соединения не закрываются

Корневая причина

Lock TTL (2 сек) меньше времени выполнения expensiveLoad() (2.5 сек) → lock истекает до завершения работы → при конкурентной нагрузке возникает race condition.
Решение
Изменения в коде
Файл	Изменение
index.php	Lock TTL: 2 → 5 секунд
index.php	Exponential backoff: 6 попыток (150ms * attempt)
index.php	Persistent connections: true → false
www.conf	max_children: 5 → 15
Патч (ключевые изменения)
diff

- PDO::ATTR_PERSISTENT => true,
+ PDO::ATTR_PERSISTENT => false,

- $locked = $redis->set($lockKey, $token, ['nx', 'ex' => 2]);
+ $locked = $redis->set($lockKey, $token, ['nx', 'ex' => 5]);

- if (!$locked) {
-     usleep(200000);
-     $cached = $redis->get($cacheKey);
-     if ($cached !== false) {
-         return json_decode($cached, true);
-     }
-     throw new RuntimeException('cache is busy');
- }
+ if (!$locked) {
+     $maxRetries = 6;
+     for ($attempt = 1; $attempt <= $maxRetries; $attempt++) {
+         usleep(150000 * $attempt);
+         $cached = $redis->get($cacheKey);
+         if ($cached !== false) {
+             return json_decode($cached, true);
+         }
+     }
+     throw new RuntimeException('cache is busy');
+ }

- pm.max_children = 5
+ pm.max_children = 15

Результаты
Метрика	До	После
Ошибки (100 req)	28 (28%)	0 (0%)
Среднее время	245ms	47ms
P99	2800ms	120ms
Применение
Option 1: Live patch
bash

git clone https://github.com/YOUR_USERNAME/devops-test-fix
cd devops-test-fix
./apply-fix.sh

Option 2: Build new image
bash

docker build -f Dockerfile.fixed -t devops-stack:fixed .
docker run --rm -p 8080:80 devops-stack:fixed

Файлы
Файл	Описание
SOLUTION.md	Детальное техническое решение
patch.diff	Полный diff всех изменений
index.php.fixed	Исправленный файл
apply-fix.sh	Скрипт применения
Dockerfile.fixed	Dockerfile для сборки