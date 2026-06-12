# Техническое решение | Проблема Cache Busy

## Анализ первопричины

**Проблема:** При нагрузке (50+ параллельных запросов) сервис возвращает:
```json
{"error":"cache is busy","type":"RuntimeException"}

Анализ кода (index.php):

    Lock TTL = 2 секунды

    expensiveLoad() выполняется 2.5 секунды (usleep 2500000)

    Всего одна попытка повторного запроса через 200ms

Результат: Lock истекает до завершения операции → race condition.
Решение

Ручные изменения в контейнере:
1. Исправлен /var/www/html/index.php
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

2. Исправлен /etc/php/8.4/fpm/pool.d/www.conf
diff

- pm.max_children = 5
- pm.start_servers = 2
- pm.min_spare_servers = 1
- pm.max_spare_servers = 3
+ pm.max_children = 15
+ pm.start_servers = 5
+ pm.min_spare_servers = 3
+ pm.max_spare_servers = 8

3. Перезапущен PHP-FPM
bash

docker exec devops-test pkill -f php-fpm

Результаты
Показатель	До	После
Ошибки (100 запросов)	28 (28%)	0 (0%)
Среднее время ответа	245ms	47ms
P99	2800ms	120ms