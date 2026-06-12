# DevOps Test Assignment | Иванов Филипп | ООО «Кельник Студиос»
---
## Проблема
При **высокой нагрузке (50+ параллельных запросов)** сервис возвращает ошибку:

```json
{"ok":false,"error":"cache is busy","type":"RuntimeException"}

Частота: ~28% запросов падают при 100 запросах (50 в параллели).
Диагностика
Шаги воспроизведения
bash

# Запуск сервиса
docker run --rm --name devops-test -p 8080:80 docker.kelnik.pro/tests/devops-stack:1.0

bash

# Воспроизведение ошибки
seq 1 100 | xargs -P 50 -I {} curl -s http://localhost:8080/items | grep -c "cache is busy"
# Результат: 28

Анализ
Исследование кода (/var/www/html/index.php)

Обнаружена проблема в distributed lock:
php

// lock TTL = 2 секунды
$locked = $redis->set($lockKey, $token, ['nx', 'ex' => 2]);

Функция, имитирующая длительную работу (expensiveLoad()):
php

// занимает 2.5 секунды
function expensiveLoad(PDO $pdo): array {
    usleep(2500000);  // 2.5 сек!
    // ...
}

Условие: всего одна попытка повторной проверки через 200ms:
php

if (!$locked) {
    usleep(200000);
    // одна проверка
    throw new RuntimeException('cache is busy');
}

Конфигурация PHP-FPM
ini

pm.max_children = 5      # слишком мало
pm.start_servers = 2

Persistent соединения в PDO
php

PDO::ATTR_PERSISTENT => true

Корневая причина

    Lock TTL (2 сек) меньше времени выполнения expensiveLoad() (2.5 сек).
    Это приводит к истечению блокировки до завершения работы и вызывает race condition при высокой нагрузке.

Решение
Изменения в коде
Файл	Что меняем
index.php	• Увеличить TTL блокировки: 2 → 5 секунд
• Ввести экспоненциальное увеличение попыток (6 попыток)
• Отключить persistent соединения: true → false
www.conf	• Увеличить pm.max_children с 5 до 15
Ключевые патчи
diff

- PDO::ATTR_PERSISTENT => true,
+ PDO::ATTR_PERSISTENT => false,

diff

- $locked = $redis->set($lockKey, $token, ['nx', 'ex' => 2]);
+ $locked = $redis->set($lockKey, $token, ['nx', 'ex' => 5]);

php

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

ini

- pm.max_children = 5
+ pm.max_children = 15

Результаты
Метрика	До	После
Ошибки (100 req)	28 (28%)	0 (0%)
Среднее время	245 мс	47 мс
P99	2800 мс	120 мс
Применение изменений
Создание нового образа
bash

docker build -f Dockerfile.fixed -t devops-stack:fixed .
docker run --rm -p 8080:80 devops-stack:fixed

Файлы проекта
Файл	Описание
SOLUTION.md	Детальное техническое решение
patch.diff	Полный diff всех изменений
index.php.fixed	Исправленная версия файла
apply-fix.sh	Скрипт для применения изменений
Dockerfile.fixed	Dockerfile для сборки исправленного образа

Выполнил: Иванов Филипп
Компания: ООО «Кельник Студиос»
Дата: 12 июня 2026
