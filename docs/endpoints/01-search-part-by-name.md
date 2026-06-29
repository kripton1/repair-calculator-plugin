# Эндпоинт: Поиск запчасти по названию

## Назначение

Поиск запчастей по текстовому запросу (название, артикул, OEM-номер). Это первый и основной эндпоинт для интеграции — пользователь вводит название детали, плагин находит совпадения с ценами и наличием.

## HTTP Request

```
POST https://api.bm.parts/v2/search
```

## Заголовки (Headers)

```
Content-Type: application/json
Authorization: Bearer {api_key}
```

📌 **Уточнить формат авторизации у BM Parts.**

## Тело запроса (Request Body)

```json
{
  "query": "тормозные колодки",
  "language_id": 1,
  "page": 1,
  "per_page": 20
}
```

### Поля запроса

| Поле | Тип | Обязательно | Описание |
|------|-----|-------------|----------|
| `query` | string | ✅ | Поисковый запрос (название детали, артикул, OEM) |
| `language_id` | integer | ❌ | ID языка ответа. 1 = українська, 2 = русский, 3 = english. По умолчанию: 1 |
| `page` | integer | ❌ | Номер страницы. По умолчанию: 1 |
| `per_page` | integer | ❌ | Количество результатов на странице. По умолчанию: 20, макс: 100 |

📌 **Уточнить:** Точные значения `language_id` и лимиты.

### Примеры запросов

```json
// Поиск по названию
{ "query": "фільтр масляний" }

// Поиск по артикулу
{ "query": "04511-SDA-A01" }

// Поиск по OEM-номеру
{ "query": "1J0698451" }
```

## Ответ (Response)

### Успешный ответ (200 OK)

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": 12345,
        "article_number": "04511-SDA-A01",
        "article_oem": "1J0698451",
        "name": "Фільтр масляний",
        "name_en": "Oil Filter",
        "brand": {
          "id": 101,
          "name": "MANN-FILTER"
        },
        "category": {
          "id": 50,
          "name": "Фільтри",
          "name_en": "Filters"
        },
        "price": {
          "retail": 450.00,
          "wholesale": 380.00,
          "currency": "UAH"
        },
        "availability": {
          "in_stock": true,
          "quantity": 25,
          "delivery_days": 1,
          "warehouse": "Київ"
        },
        "supplier": {
          "id": 201,
          "name": "BM Parts"
        },
        "image_url": "https://bm.parts/images/parts/04511-SDA-A01.jpg",
        "description": "Оригінальний масляний фільтр для Honda Civic"
      }
    ],
    "pagination": {
      "current_page": 1,
      "per_page": 20,
      "total_items": 45,
      "total_pages": 3
    }
  }
}
```

📌 **Уточнить:** Точную структуру ответа по документации BM Parts API.

### Поля ответа

| Поле | Тип | Описание |
|------|-----|----------|
| `data.items[].id` | integer | Уникальный ID детали |
| `data.items[].article_number` | string | Артикульный номер производителя |
| `data.items[].article_oem` | string | OEM-номер (номер оригинальной детали) |
| `data.items[].name` | string | Название на языке запроса |
| `data.items[].brand.name` | string | Бренд/производитель |
| `data.items[].category.name` | string | Категория детали |
| `data.items[].price.retail` | float | Розничная цена |
| `data.items[].price.wholesale` | float | Оптовая цена |
| `data.items[].price.currency` | string | Валюта (UAH) |
| `data.items[].availability.in_stock` | boolean | В наличии |
| `data.items[].availability.quantity` | integer | Количество на складе |
| `data.items[].availability.delivery_days` | integer | Дни до доставки |

### Обработка ошибок

| Код | Ответ | Описание |
|-----|-------|----------|
| 400 | `{ "success": false, "error": "Invalid query parameter" }` | Невалидный запрос |
| 401 | `{ "success": false, "error": "Unauthorized" }` | Невалидный API-ключ |
| 403 | `{ "message": "Превышен лимит запросов API для ..." }` | Rate limit — смотреть `X-RateLimit-Reset` |
| 429 | `{ "success": false, "error": "Rate limit exceeded" }` | Лимит запросов |
| 500 | `{ "success": false, "error": "Internal server error" }` | Ошибка сервера |

### Rate Limit в реальном времени

```php
// Извлечение лимитов из заголовков ответа
$headers = wp_remote_retrieve_headers($response);
$remaining = $headers['x-ratelimit-remaining'] ?? null;
$reset     = $headers['x-ratelimit-reset'] ?? null;

if ($remaining !== null && $remaining < 100) {
    error_log("BM Parts API: осталось {$remaining} запросов, сброс через {$reset}");
}
```

---

## Реализация в плагине

### PHP: Метод поиска

```php
class BM_Parts_API {

    /**
     * Поиск запчасти по названию/артикулу
     *
     * @param string $query     Поисковый запрос
     * @param int    $page      Номер страницы
     * @param int    $perPage   Результатов на странице
     * @return array Результаты поиска
     */
    public function search_parts(
        string $query,
        int $page = 1,
        int $perPage = 20
    ): array {
        $cache_key = 'bmparts_search_' . md5($query . $page . $perPage);
        $cached = get_transient($cache_key);

        if ($cached !== false) {
            return $cached;
        }

        $data = $this->request('POST', 'search', [
            'query'       => sanitize_text_field($query),
            'language_id' => (int) get_option('bmparts_language_id', 1),
            'page'        => $page,
            'per_page'    => min($perPage, 100),
        ]);

        // Кэшируем на 1 час
        set_transient($cache_key, $data, HOUR_IN_SECONDS);

        return $data;
    }
}
```

### WP REST API: Прокси-эндпоинт

```php
// Регистрация REST API маршрута
add_action('rest_api_init', function () {
    register_rest_route('repair-calculator/v1', '/search', [
        'methods'             => 'POST',
        'callback'            => 'rc_search_parts',
        'permission_callback' => '__return_true', // Публичный доступ
        'args' => [
            'query' => [
                'required'          => true,
                'sanitize_callback' => 'sanitize_text_field',
                'validate_callback' => function ($param) {
                    return is_string($param) && strlen($param) >= 2;
                },
            ],
        ],
    ]);
});

function rc_search_parts(WP_REST_Request $request): WP_REST_Response {
    $query    = $request->get_param('query');
    $page     = (int) $request->get_param('page') ?: 1;
    $per_page = (int) $request->get_param('per_page') ?: 20;

    $api = new BM_Parts_API();

    try {
        $result = $api->search_parts($query, $page, $per_page);
        return new WP_REST_Response($result, 200);
    } catch (Exception $e) {
        return new WP_REST_Response([
            'success' => false,
            'error'   => $e->getMessage(),
        ], 500);
    }
}
```

### JavaScript: Фронтенд-вызов

```javascript
// assets/js/calculator.js

class RepairCalculator {
    constructor() {
        this.searchInput = document.getElementById('rc-search-input');
        this.resultsContainer = document.getElementById('rc-results');
        this.debounceTimer = null;

        this.init();
    }

    init() {
        this.searchInput.addEventListener('input', (e) => {
            clearTimeout(this.debounceTimer);
            this.debounceTimer = setTimeout(() => {
                this.search(e.target.value);
            }, 500); // debounce 500ms
        });
    }

    async search(query) {
        if (query.length < 2) {
            this.resultsContainer.innerHTML = '';
            return;
        }

        try {
            const response = await fetch('/wp-json/repair-calculator/v1/search', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ query }),
            });

            const data = await response.json();

            if (data.success) {
                this.renderResults(data.data.items);
            }
        } catch (error) {
            console.error('Search error:', error);
        }
    }

    renderResults(items) {
        this.resultsContainer.innerHTML = items.map(item => `
            <div class="rc-result-item" data-id="${item.id}">
                <div class="rc-result-name">${item.name}</div>
                <div class="rc-result-brand">${item.brand.name}</div>
                <div class="rc-result-article">${item.article_number}</div>
                <div class="rc-result-price">${item.price.retail} ${item.price.currency}</div>
                <div class="rc-result-availability">
                    ${item.availability.in_stock
                        ? `✅ В наличии (${item.availability.quantity} шт.)`
                        : `❌ Нет в наличии`
                    }
                </div>
                <button class="rc-btn-add" onclick="calculator.addPart(${item.id})">
                    + Добавить
                </button>
            </div>
        `).join('');
    }
}

const calculator = new RepairCalculator();
```

---

## Тестирование

### cURL

```bash
# Поиск по названию
curl -i https://api.bm.parts/v2/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "query": "тормозные колодки",
    "language_id": 2,
    "page": 1,
    "per_page": 10
  }'

# Поиск по артикулу
curl -i https://api.bm.parts/v2/search \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "query": "04511-SDA-A01"
  }'
```

### Тестовые сценарии

| # | Сценарий | Ожидаемый результат |
|---|----------|-------------------|
| 1 | Поиск "фільтр масляний" | Список масляных фильтров с ценами |
| 2 | Поиск по артикулу "04511-SDA-A01" | Одна точная деталь |
| 3 | Поиск с пустым запросом | Ошибка 400 |
| 4 | Поиск с query < 2 символов | Ошибка валидации |
| 5 | Поиск несуществующей детали | Пустой список `items: []` |

---

## TODO

- [x] Лимиты rate limiting (10 000/час авториз., 120/час неавториз.)
- [ ] Уточнить формат авторизации (Bearer / API Key / другой)
- [ ] Уточнить структуру ответа (поля, типы)
- [ ] Уточнить language_id для русского языка
- [ ] Протестировать с реальным API-ключом
