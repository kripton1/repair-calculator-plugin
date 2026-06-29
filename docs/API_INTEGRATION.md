# Интеграция с BM Parts API v2

## Общая информация

| Параметр | Значение |
|----------|----------|
| Base URL | `https://api.bm.parts/` |
| Протокол | HTTPS |
| Формат данных | JSON |
| Авторизация | API-ключ (сессия или ключ) |

### Лимиты запросов

| Тип | Лимит | Привязка |
|-----|-------|----------|
| Авторизованные | **10 000 запросов/час** | Контрагент пользователя (все приложения/токены делят квоту) |
| Неавторизованные | **120 запросов/час** | IP-адрес |

> ⚠️ API находится в режиме активной разработки. Методы и ответы могут изменяться.

> **Важно:** Никогда не передавайте API-ключ в публичной части приложения. Используйте его только при сервер-сервер взаимодействии.

## Доступ к API

Для работы с API необходим аккаунт на [BM Parts](https://bm.parts):
1. Зарегистрироваться на [busmarket.group/ua/for-clients](https://busmarket.group/ua/for-clients)
2. Получить API-ключ (credentials)
3. Сохранить ключ в настройках плагина

## Архитектура интеграции

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  WP Frontend    │────▶│  WP REST API     │────▶│  BM Parts API   │
│  (Calculator)   │     │  (Proxy/Backend) │     │  v2             │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                              │
                         ┌────┴────┐
                         │ Cache   │
                         │(Transients)│
                         └─────────┘
```

**Почему проксируем через WP REST API:**
- Скрытие API-ключа BM Parts от клиента
- Кэширование результатов
- Контроль rate-limiting
- Обработка ошибок на сервере

## Класс-клиент: `BM_Parts_API`

```php
class BM_Parts_API {
    private string $base_url = 'https://developer.bm.parts/api/v2/';
    private string $api_key;
    private int $cache_ttl = 3600; // 1 час

    public function __construct() {
        $this->api_key = get_option('bmparts_api_key', '');
    }

    /**
     * Универсальный запрос к API
     */
    private function request(string $method, string $endpoint, array $data = []): array {
        $url = $this->base_url . $endpoint;

        $args = [
            'method'  => $method,
            'timeout' => 30,
            'headers' => [
                'Content-Type'  => 'application/json',
                'Authorization' => 'Bearer ' . $this->api_key,
                // 📌 Уточнить формат авторизации
            ],
        ];

        if (!empty($data)) {
            $args['body'] = wp_json_encode($data);
        }

        $response = wp_remote_request($url, $args);

        if (is_wp_error($response)) {
            throw new Exception('BM Parts API error: ' . $response->get_error_message());
        }

        $body = wp_remote_retrieve_body($response);
        $code = wp_remote_retrieve_response_code($response);

        if ($code >= 400) {
            throw new Exception("BM Parts API HTTP {$code}: {$body}");
        }

        return json_decode($body, true) ?? [];
    }

    // ... методы эндпоинтов ниже
}
```

## Доступные эндпоинты (из описания портала)

Согласно описанию на `developer.bm.parts`, API v2 содержит:

| Функционал | Описание |
|-----------|----------|
| Поиск запчастей | 🔵 Разработка |
| Создание корзин | ⏳ Этап 2 |
| Резервирование | ⏳ Этап 3 |
| Отгрузка товара | ⏳ Этап 4 |
| Документация/отчёты | ⏳ Этап 5 |

## Кэширование

```php
// Пример кэширования через Transients
function cached_search($query) {
    $cache_key = 'bmparts_search_' . md5($query);
    $cached = get_transient($cache_key);

    if ($cached !== false) {
        return $cached;
    }

    $result = $api->search_parts($query);
    set_transient($cache_key, $result, HOUR_IN_SECONDS);

    return $result;
}
```

## Обработка ошибок

| Код | Описание | Действие |
|-----|----------|---------|
| 401 | Неавторизован | Проверить API-ключ в настройках |
| 403 | Доступ запрещён | Проверить права аккаунта |
| 429 | Rate limit | Увеличить задержку, показать кэш |
| 500 | Серверная ошибка | Логировать, показать сообщение |

## Rate Limiting

Лимиты определены API: 10 000/час (авторизованные), 120/час (неавторизованные).

### Заголовки ответа

API возвращает текущий статус лимита в каждом ответе:

```
X-RateLimit-Limit: 10000
X-RateLimit-Remaining: 9950
X-RateLimit-Reset: 1372700873  # Unix timestamp UTC
```

### Обработка 43 (Rate Limit)

```json
HTTP/1.1 403 Forbidden
X-RateLimit-Limit: 10000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1377013266

{
  "message": "Превышен лимит запросов API для xxx.xxx.xxx.xxx.",
  "documentation_url": "https://developer.bm.parts/"
}
```

### Стратегия работы с лимитами

1. **Кэширование** — ответы кэшируются через Transients (TTL 1 час)
2. **Debounce** — фронтенд-запросы с задержкой 500ms
3. **Мониторинг** — парсинг `X-RateLimit-Remaining` для предупреждений
4. **Fallback** — при 403 показывать кэшированные данные вместо ошибки

## Следующий шаг

→ [01-search-part-by-name.md](./endpoints/01-search-part-by-name.md) — Эндпоинт поиска запчасти по названию
