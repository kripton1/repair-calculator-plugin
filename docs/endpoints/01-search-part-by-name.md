# Эндпоинт: Поиск запчастей по названию

> Источник: [developer.bm.parts/api/v2/search_products.html](https://developer.bm.parts/api/v2/search_products.html)

## Назначение

Поиск запчастей по текстовому запросу (название, артикул, OEM-номер). Это первый и основной эндпоинт для интеграции.

## Base URL

```
GET https://api.bm.parts/search/products
```

⚠️ **Это GET-запрос с query-параметрами, не POST с JSON-телом!**

## Заголовки (Headers)

```
Authorization: {api_key}
User-Agent: RepairCalculatorWP/1.0
Accept-Language: ru
```

| Заголовок | Обязательно | Описание |
|-----------|-------------|----------|
| `Authorization` | ✅ | API-ключ **без префикса Bearer** |
| `User-Agent` | ✅ | **Обязателен!** Без него — 403 |
| `Accept-Language` | ❌ | `ru` или `uk` (по умолчанию: `uk`) |

## Параметры запроса (Query Parameters)

### Основные

| Параметр | Тип | Обязательно | Описание |
|----------|-----|-------------|----------|
| `q` | string | ✅ | Поисковый запрос: `&q=амортизатор передній` или `&q=115 906` |
| `search_mode` | string | ❌ | Тип поиска: `strict` (точный), `partial` (частичный), `extended` (расширенный, по умолчанию) |
| `page` | integer | ❌ | Номер страницы (с 1). По умолчанию: 1 |
| `per_page` | integer | ❌ | Элементов на странице (до 100). По умолчанию: 30 |
| `save` | boolean | ❌ | Сохранять в историю поиска. По умолчанию: `true` (требует `q`) |
| `products_as` | string | ❌ | Формат товаров: `obj` (по умолчанию) или `arr` |

### Фильтры

| Параметр | Тип | Описание | Пример |
|----------|-----|----------|--------|
| `available` | 1/0 | Только в наличии | `&available=1` |
| `brands` | string[] | Фильтр по брендам (множественные) | `&brands=SACHS&brands=SOLGY` |
| `group` | string | Фильтр по товарной группе (path) | `&group=ДВИГАТЕЛЬ` |
| `nodes` | string | Фильтр по сборочному узлу | `&nodes=Легковые автомобили/Подвеска` |
| `end_nodes` | string | Фильтр по быстрой группе | `&end_nodes=Кронштейн крепления глушника` |
| `gf` | string | Параметры группы каталога | `&gf=Максимальная напряжение:58V` |
| `cars` | string[] | Фильтр по авто (маркa>модель) | `&cars=BMW&cars=MERCEDES-BENZ>SPRINTER 2-t` |
| `new_product` | 1/0 | Только новинки | `&new_product=1` |
| `promo` | 1/0 | Только акции | `&promo=1` |
| `special_offer` | 1/0 | Только спецпредложения | `&special_offer=1` |
| `tools` | 1/0 | Только инструменты | `&tools=1` |
| `badges` | string[] | Бейджи: `promo`, `seasonal`, `top`, `sale`, `new` | `&badges=new&badges=sale` |
| `advertisement` | string | ID рекламной акции | `&advertisement=UUID` |
| `new_arrival` | string | ID нового поступления | `&new_arrival=UUID` |
| `warehouses` | string[] | ID складов для остатков. `all` — все склады | `&warehouses=UUID1&warehouses=UUID2` |
| `currency` | string | Валюта цен (ID валюты договора) | `&currency=UUID` |
| `no_discounts` | 1/0 | Без учёта индивидуальных скидок | `&no_discounts=1` |

> ⚠️ Если марка/модель содержит `/`, нужно двойное кодирование: `encodeURIComponent(encodeURIComponent("FORD ASIA / OZEANIA"))`

## Примеры запросов

```bash
# Простой поиск по названию
GET https://api.bm.parts/search/products?q=амортизатор

# Поиск по артикулу + бренд
GET https://api.bm.parts/search/products?q=115+906&brands=SACHS

# Поиск по авто + бренд
GET https://api.bm.parts/search/products?brands=SOLGY&cars=MERCEDES-BENZ>SPRINTER+2-t

# С фильтрами наличия и складов
GET https://api.bm.parts/search/products?q=115+906+амортизатор&brands=SACHS
  &available=1
  &warehouses=816D000C2999A7E611E6EC6B4A1915AF
  &warehouses=ACF9000C2947F7AE11E28A2B02C4AD32
  &currency=A358000C2947F7AE11E23F5617780B16
  &page=2

# Поиск по сборочному узлу + авто
GET https://api.bm.parts/search/products?q=Подушка+двигателя&brands=TURK
  &nodes=Легковые автомобили/ДВИГАТЕЛЬ
  &cars=MAZDA
```

## Ответ (Response)

### Формат `products_as=obj` (по умолчанию)

```json
{
  "products": {
    "9D909508CD2293B6499E68280D31A49B": {
      "in_wishlist": true,
      "name": "Амортизатор (задний) Ford Kuga 08-",
      "default_image": "https://cdn.bm.parts/photo/52x52/w/d/a/d/d/e393c33f.jpeg",
      "brand": "SACHS",
      "article": "115 906",
      "promotion": false,
      "promos": [],
      "new_product": false,
      "special_offer": false,
      "tools": false,
      "badges": ["promo"],
      "highlight": null,
      "price": 1103.836,
      "currency_name": "ГРН",
      "available": true,
      "in_cart": null,
      "found_by": null,
      "multiplicity": 1,
      "restricted_delivery": null,
      "in_stocks": [
        {
          "uuid": "816D000C2999A7E611E6EC6B4A1915AF",
          "name": "Київ Борщагівка",
          "short_name": null,
          "has_reserves": false,
          "quantity": "-",
          "contract_quantities": [
            { "f": 1, "quantity": "-" }
          ]
        },
        {
          "uuid": "ACF9000C2947F7AE11E28A2B02C4AD32",
          "name": "Луцьк",
          "short_name": "Лцк",
          "has_reserves": false,
          "quantity": "2",
          "contract_quantities": [
            { "f": 1, "quantity": "2" }
          ]
        }
      ],
      "in_others": {
        "uuid": "-",
        "name": "На других",
        "has_reserves": false,
        "quantity": "-"
      },
      "in_waiting": {
        "uuid": "-",
        "name": "Ожидается",
        "quantity": "-"
      }
    }
  },
  "headers": [
    {
      "warehouse": "Луцьк",
      "short_name": "Лцк",
      "labels": ["ДАГ"]
    }
  ],
  "search": {
    "hits": 1,
    "total_pages": 1
  }
}
```

### Формат `products_as=arr`

При `products_as=arr` товары возвращаются как массив вместо объекта.

### Поля товара

| Поле | Тип | Описание |
|------|-----|----------|
| `name` | string | Название товара |
| `brand` | string | Бренд |
| `article` | string | Артикульный номер |
| `uuid` | string | Уникальный UUID товара |
| `price` | float | Цена |
| `currency_name` | string | Валюта (ГРН) |
| `available` | boolean | В наличии |
| `promotion` | boolean | Акционный товар |
| `new_product` | boolean | Новинка |
| `special_offer` | boolean | Спецпредложение |
| `tools` | boolean | Инструмент |
| `badges` | string[] | Бейджи товара |
| `default_image` | string | URL изображения |
| `multiplicity` | integer | Кратность заказа |
| `found_by` | string | Что совпало при поиске |
| `highlight` | string | Подсветка совпадений |
| `in_stocks` | array | Остатки по складам |
| `in_others` | object | Остатки на других складах |
| `in_waiting` | object | Ожидаемые поступления |

### Поля склада (`in_stocks[]`)

| Поле | Тип | Описание |
|------|-----|----------|
| `uuid` | string | UUID склада |
| `name` | string | Название склада |
| `short_name` | string | Краткое название |
| `has_reserves` | boolean | Есть резервы |
| `quantity` | string | Количество: `"-"`, `"2"`, `"> 20"` |
| `contract_quantities` | array | Количество по договорам |

### Поля пагинации (`search`)

| Поле | Тип | Описание |
|------|-----|----------|
| `hits` | integer | Общее количество найденных товаров |
| `total_pages` | integer | Общее количество страниц |

### Поля заголовков складов (`headers[]`)

| Поле | Тип | Описание |
|------|-----|----------|
| `warehouse` | string | Название склада |
| `short_name` | string | Краткое название |
| `labels` | string[] | Метки склада (например, `["ДАГ"]`) |

---

## Дополнительные эндпоинты поиска

### Подсказка поиска (suggest)

```
GET /search/products/suggest?q={query}
```

Возвращает подсказки для поискового запроса. Полезно для autocomplete.

### Поиск по UUID

```
POST /search/products
Body: { "product_uuids": ["UUID1", "UUID2"] }
```

Получение информации о конкретных товарах по UUID.

### Подсказка (search/suggests)

```
GET /search/suggests?q={query}
```

Возвращает `type: "products"` или `type: "vehicle"` (если запрос — VIN).

### Дерево товарных групп

```
GET /search/products/groups
```

Возвращает иерархию групп товаров (ДВИГАТЕЛЬ → Кривошипно-шатунный механизм → ...).

### Агрегация по брендам

```
GET /search/products/aggregations/brands
```

### Агрегация по моделям для марки

```
GET /search/products/aggregations/car/{car_name}/models
```

### Агрегация по двигателям для модели

```
GET /search/products/aggregations/car/{car_name}/model/{model_name}
```

### История поиска

```
GET /search/products/history
```

Возвращает последние 10 результатов поиска (без параметров).

---

## Реализация в плагине

### PHP: Метод поиска

```php
class BM_Parts_API {

    /**
     * Поиск запчастей по названию/артикулу
     *
     * @param string $query       Поисковый запрос
     * @param array  $filters     Дополнительные фильтры
     * @param int    $page        Номер страницы
     * @param int    $perPage     Результатов на странице
     * @return array Результаты поиска
     */
    public function search_products(
        string $query,
        array  $filters = [],
        int    $page = 1,
        int    $perPage = 30
    ): array {
        $cache_key = 'bmparts_search_' . md5($query . json_encode($filters) . $page . $perPage);
        $cached = get_transient($cache_key);

        if ($cached !== false) {
            return $cached;
        }

        $params = array_merge([
            'q'          => $query,
            'page'       => $page,
            'per_page'   => min($perPage, 100),
            'products_as' => 'obj',
        ], $filters);

        $data = $this->request('GET', 'search/products', $params);

        set_transient($cache_key, $data, HOUR_IN_SECONDS);

        return $data;
    }

    /**
     * Подсказка поиска (autocomplete)
     */
    public function suggest(string $query): array {
        return $this->request('GET', 'search/products/suggest', [
            'q' => $query,
        ]);
    }

    /**
     * Дерево товарных групп
     */
    public function product_groups(): array {
        return $this->request('GET', 'search/products/groups');
    }

    /**
     * Агрегация по брендам для поискового запроса
     */
    public function aggregation_brands(array $params = []): array {
        return $this->request('GET', 'search/products/aggregations/brands', $params);
    }

    /**
     * Агрегация по моделям для марки авто
     */
    public function aggregation_models(string $car_name): array {
        $car_encoded = rawurlencode($car_name);
        return $this->request('GET', "search/products/aggregations/car/{$car_encoded}/models");
    }

    /**
     * Агрегация по двигателям для модели
     */
    public function aggregation_engines(string $car_name, string $model_name): array {
        $car_encoded    = rawurlencode($car_name);
        $model_encoded  = rawurlencode($model_name);
        return $this->request('GET',
            "search/products/aggregations/car/{$car_encoded}/model/{$model_encoded}");
    }
}
```

### PHP: Универсальный HTTP-клиент

```php
private function request(string $method, string $endpoint, array $params = []): array {
    $url = $this->base_url . '/' . ltrim($endpoint, '/');

    $args = [
        'method'  => $method,
        'timeout' => 30,
        'headers' => [
            'Authorization'   => $this->api_key,
            'User-Agent'      => 'RepairCalculatorWP/1.0',
            'Accept-Language' => get_option('bmparts_lang', 'ru'),
        ],
    ];

    // GET — параметры в URL, POST — в body
    if ($method === 'GET' && !empty($params)) {
        $url = add_query_arg($params, $url);
    } elseif ($method === 'POST' && !empty($params)) {
        $args['headers']['Content-Type'] = 'application/json';
        $args['body'] = wp_json_encode($params);
    }

    $response = wp_remote_request($url, $args);

    if (is_wp_error($response)) {
        throw new Exception('BM Parts API error: ' . $response->get_error_message());
    }

    $code    = wp_remote_retrieve_response_code($response);
    $body    = wp_remote_retrieve_body($response);
    $headers = wp_remote_retrieve_headers($response);

    // Rate limit мониторинг
    $remaining = $headers['x-ratelimit-remaining'] ?? null;
    if ($remaining !== null && $remaining < 100) {
        error_log("BM Parts API: осталось {$remaining} запросов");
    }

    if ($code === 403 && isset($headers['x-ratelimit-remaining'])) {
        throw new Exception('BM Parts API: превышен лимит запросов');
    }

    if ($code >= 400) {
        throw new Exception("BM Parts API HTTP {$code}: {$body}");
    }

    return json_decode($body, true) ?? [];
}
```

### WP REST API: Прокси-эндпоинт

```php
add_action('rest_api_init', function () {
    register_rest_route('repair-calculator/v1', '/search', [
        'methods'             => 'GET',
        'callback'            => 'rc_search_parts',
        'permission_callback' => '__return_true',
        'args' => [
            'q' => [
                'required'          => true,
                'sanitize_callback' => 'sanitize_text_field',
            ],
        ],
    ]);
});

function rc_search_parts(WP_REST_Request $request): WP_REST_Response {
    $query = $request->get_param('q');
    $page  = (int) $request->get_param('page') ?: 1;

    // Фильтры из запроса
    $filters = [];
    if ($request->get_param('brands')) {
        $filters['brands'] = $request->get_param('brands');
    }
    if ($request->get_param('available')) {
        $filters['available'] = 1;
    }
    if ($request->get_param('cars')) {
        $filters['cars'] = $request->get_param('cars');
    }

    $api = new BM_Parts_API();

    try {
        $result = $api->search_products($query, $filters, $page);
        return new WP_REST_Response($result, 200);
    } catch (Exception $e) {
        return new WP_REST_Response([
            'success' => false,
            'error'   => $e->getMessage(),
        ], 500);
    }
}
```

### JavaScript: Фронтенд

```javascript
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
            }, 500);
        });
    }

    async search(query) {
        if (query.length < 2) {
            this.resultsContainer.innerHTML = '';
            return;
        }

        try {
            const params = new URLSearchParams({ q: query });
            const response = await fetch(`/wp-json/repair-calculator/v1/search?${params}`);
            const data = await response.json();

            if (data.products) {
                this.renderResults(data.products, data.search);
            }
        } catch (error) {
            console.error('Search error:', error);
        }
    }

    renderResults(products, pagination) {
        const items = Object.entries(products).map(([uuid, product]) => `
            <div class="rc-result-item" data-uuid="${uuid}">
                <img src="${product.default_image}" alt="${product.name}" class="rc-result-image">
                <div class="rc-result-info">
                    <div class="rc-result-name">${product.name}</div>
                    <div class="rc-result-brand">${product.brand}</div>
                    <div class="rc-result-article">Артикул: ${product.article}</div>
                    <div class="rc-result-price">${product.price} ${product.currency_name}</div>
                    <div class="rc-result-availability">
                        ${product.available ? '✅ В наличии' : '❌ Нет в наличии'}
                    </div>
                </div>
                <button class="rc-btn-add" onclick="calculator.addPart('${uuid}')">
                    + Добавить
                </button>
            </div>
        `).join('');

        this.resultsContainer.innerHTML = `
            <div class="rc-results-count">Найдено: ${pagination?.hits || 0}</div>
            ${items}
        `;
    }
}

const calculator = new RepairCalculator();
```

---

## Тестирование

### cURL

```bash
# Простой поиск
curl -i "https://api.bm.parts/search/products?q=амортизатор" \
  -H "Authorization: YOUR_API_KEY" \
  -H "User-Agent: RepairCalculatorWP/1.0" \
  -H "Accept-Language: ru"

# Поиск по артикулу + бренд + наличие
curl -i "https://api.bm.parts/search/products?q=115+906&brands=SACHS&available=1" \
  -H "Authorization: YOUR_API_KEY" \
  -H "User-Agent: RepairCalculatorWP/1.0"

# Подсказка поиска
curl -i "https://api.bm.parts/search/products/suggest?q=амортизатор" \
  -H "Authorization: YOUR_API_KEY" \
  -H "User-Agent: RepairCalculatorWP/1.0"

# Дерево товарных групп
curl -i "https://api.bm.parts/search/products/groups" \
  -H "Authorization: YOUR_API_KEY" \
  -H "User-Agent: RepairCalculatorWP/1.0"
```

### Тестовые сценарии

| # | Сценарий | Ожидаемый результат |
|---|----------|-------------------|
| 1 | `?q=амортизатор` | Список амортизаторов с ценами |
| 2 | `?q=115+906&brands=SACHS` | Точная деталь SACHS |
| 3 | `?q=амортизатор&available=1` | Только в наличии |
| 4 | `?q=амортизатор&cars=MERCEDES-BENZ>SPRINTER+2-t` | Только для SPRINTER |
| 5 | Пустой `q` | Ошибка или пустой результат |
| 6 | Без User-Agent | 403 Forbidden |
| 7 | Превышен rate limit | 403 + X-RateLimit-* заголовки |
| 8 | `?q=WV1ZZZ2KZ7X006849` (VIN) | suggest → type: "vehicle" |

---

## TODO

- [x] Формат авторизации: `Authorization: API-КЛЮЧ`
- [x] Обязательный `User-Agent`
- [x] GET-запрос с query-параметрами (не POST JSON)
- [x] Пагинация: `page`, `per_page` (до 100)
- [x] Локализация: `Accept-Language: ru`/`uk`
- [x] Фильтры: brands, cars, available, nodes, badges, etc.
- [x] Ответ: products (obj/arr), headers, search (hits, total_pages)
- [ ] Протестировать с реальным API-ключом
