# Интеграция с BM Parts API v2

> Источник: [developer.bm.parts/api/v2/overview.html](https://developer.bm.parts/api/v2/overview.html)

## Общая информация

| Параметр | Значение |
|----------|----------|
| Base URL | `https://api.bm.parts` |
| Протокол | HTTPS |
| Формат данных | JSON |
| Даты | ISO 8601: `YYYY-MM-DDTHH:MM:SSZ` |

## Аутентификация

Два метода аутентификации:

### 1. API-ключ (рекомендуется)

Передаётся в заголовке `Authorization`. **Формат: `Authorization: API-КЛЮЧ`** (без префикса `Bearer`).

```bash
curl -H "Authorization: YOUR_API_KEY" https://api.bm.parts/profile/me
```

- Ключ генерируется на [login.bm.parts/api/](https://login.bm.parts/api/)
- Срок жизни ключа: **1 год** с момента генерации
- ⚠️ Никогда не передавайте ключ на фронтенд — только сервер-сервер

### 2. Сессия (cookie)

Для корпоративных доменов BM Parts — сквозная аутентификация через cookie.

```bash
curl -H "cookie: session=5c682932-7c03-4abe-854c-febd8f5ef848..." https://api.bm.parts/profile/me
```

### ⚠️ Особенность

Часть запросов может возвращать **404 Not Found вместо 403 Forbidden** для сокрытия данных от неавторизованных пользователей.

## Обязательный заголовок User-Agent

**Все запросы ОБЯЗАНЫ содержать заголовок `User-Agent`.** Без него — 403 Forbidden.

```php
$args['headers']['User-Agent'] = 'RepairCalculatorWP/1.0';
```

## Пагинация

| Параметр | Описание | По умолчанию |
|----------|----------|--------------|
| `?page=N` | Номер страницы (с 1) | 1 |
| `?per_page=N` | Элементов на странице (до 100) | 30 |

```bash
curl 'https://api.bm.parts/documents/list?page=2&per_page=100'
```

## Локалізація

Определяется заголовком `Accept-Language`:

| Значение | Язык |
|----------|------|
| `uk` | Українська (по умолчанию) |
| `ru` | Русский |

```
Accept-Language: ru
```

## Сортировка

Параметр `direction`:

| Значение | Описание |
|----------|----------|
| `desc` | Обратная сортировка (по умолчанию) |
| `asc` | Прямая сортировка |

## Указание периода

Для запросов с фильтрацией по дате — параметр `period`:

```bash
# Конкретные даты
&period={2018-02-23T00:00:01Z,2018-02-28T23:59:59Z}

# Без времени (начало/конец дня)
&period={2018-02-01,2018-01-31}

# Зарезервированные слова
&period={today}
&period={week}
&period={month}
```

| Слово | Описание |
|-------|----------|
| `today` | Сегодня |
| `yesterday` | Вчера |
| `week` | Текущая неделя |
| `month` | Текущий месяц |
| `quarter` | Текущий квартал |
| `year` | Текущий год |
| `all` | Весь период |

## Rate Limiting

| Тип | Лимит | Привязка |
|-----|-------|----------|
| Авторизованные | **10 000 запросов/час** | Контрагент пользователя (все приложения/токены делят квоту) |
| Неавторизованные | **120 запросов/час** | IP-адрес |

### Заголовки ответа

```
X-RateLimit-Limit: 10000
X-RateLimit-Remaining: 4987
X-RateLimit-Reset: 1350085394  # Unix timestamp UTC
```

### Ошибка при превышении

```http
HTTP/1.1 403 Forbidden
X-RateLimit-Limit: 120
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1377013266

{
  "message": "Превышен лимит запросов API для xxx.xxx.xxx.xxx.",
  "documentation_url": "https://developer.bm.parts/"
}
```

### Стратегия

1. **Кэширование** — Transients API (TTL 1 час)
2. **Debounce** — фронтенд 500ms
3. **Мониторинг** — парсинг `X-RateLimit-Remaining`
4. **Fallback** — при 403 показывать кэш

## CORS

API поддерживает CORS для любых доменов. Доступные заголовки:

```
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: Authorization, Content-Type, If-Match, If-Modified-Since,
                              If-None-Match, If-Unmodified-Since, X-Requested-With
Access-Control-Allow-Methods: GET, POST, PATCH, PUT, DELETE
Access-Control-Expose-Headers: ETag, X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset
```

> Это значит, что теоретически можно вызывать API прямо из браузера. Но **не делаем этого** — API-ключ не должен попасть на клиент.

## Архитектура интеграции

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  WP Frontend    │────▶│  WP REST API     │────▶│  BM Parts API   │
│  (Calculator)   │     │  (Proxy/Backend) │     │  api.bm.parts   │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                              │
                         ┌────┴────┐
                         │  Cache  │
                         │(Transients)│
                         └─────────┘
```

## Класс-клиент: `BM_Parts_API`

```php
class BM_Parts_API {
    private string $base_url = 'https://api.bm.parts';
    private string $api_key;

    public function __construct() {
        $this->api_key = get_option('bmparts_api_key', '');
    }

    /**
     * Универсальный запрос к API
     */
    private function request(string $method, string $endpoint, array $data = []): array {
        $url = $this->base_url . '/' . ltrim($endpoint, '/');

        $args = [
            'method'  => $method,
            'timeout' => 30,
            'headers' => [
                'Content-Type'   => 'application/json',
                'Authorization'  => $this->api_key,          // Формат: API-КЛЮЧ (без Bearer)
                'User-Agent'     => 'RepairCalculatorWP/1.0',
                'Accept-Language' => get_option('bmparts_lang', 'ru'),
            ],
        ];

        if (!empty($data)) {
            $args['body'] = wp_json_encode($data);
        }

        $response = wp_remote_request($url, $args);

        if (is_wp_error($response)) {
            throw new Exception('BM Parts API error: ' . $response->get_error_message());
        }

        $code    = wp_remote_retrieve_response_code($response);
        $body    = wp_remote_retrieve_body($response);
        $headers = wp_remote_retrieve_headers($response);

        // Мониторинг rate limit
        $remaining = $headers['x-ratelimit-remaining'] ?? null;
        if ($remaining !== null && $remaining < 100) {
            error_log("BM Parts API: осталось {$remaining} запросов");
        }

        if ($code === 403 && isset($headers['x-ratelimit-remaining'])) {
            throw new Exception('BM Parts API: превышен лимит запросов. Сброс через '
                . $headers['x-ratelimit-reset']);
        }

        if ($code >= 400) {
            throw new Exception("BM Parts API HTTP {$code}: {$body}");
        }

        return json_decode($body, true) ?? [];
    }

    // ... методы эндпоинтов
}
```

## Доступные эндпоинты (из описания портала)

| Функционал | Описание | Статус |
|-----------|----------|--------|
| Поиск запчастей | 🔵 Разработка | → [01-search-part-by-name.md](./endpoints/01-search-part-by-name.md) |
| Создание корзин | ⏳ Этап 2 | |
| Резервирование | ⏳ Этап 3 | |
| Отгрузка товара | ⏳ Этап 4 | |
| Документация/отчёты | ⏳ Этап 5 | |

## Следующий шаг

→ [01-search-part-by-name.md](./endpoints/01-search-part-by-name.md) — Эндпоинт поиска запчасти по названию
