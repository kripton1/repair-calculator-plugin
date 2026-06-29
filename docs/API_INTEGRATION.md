# Интеграция с BM Parts API v2

## Общая информация

| Параметр | Значение |
|----------|----------|
| Base URL | `https://developer.bm.parts/api/v2/` |
| Протокол | HTTPS |
| Формат данных | JSON |
| Авторизация | 📌 **Уточнить** (Bearer Token / API Key) |

> ⚠️ API находится в режиме активной разработки. Методы и ответы могут изменяться.

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

📌 **Уточнить у BM Parts:** лимиты запросов в минуту/час.

Рекомендуемая стратегия:
- Кэшировать результаты поиска на 1 час
- Ограничивать фронтенд-запросы debounce 500ms
- Не более 1 запроса в секунду к API

## Следующий шаг

→ [01-search-part-by-name.md](./endpoints/01-search-part-by-name.md) — Эндпоинт поиска запчасти по названию
