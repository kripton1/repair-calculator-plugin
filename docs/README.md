# Калькулятор стоимости ремонта — WordPress Plugin

## Обзор

WordPress-плагин, который позволяет пользователю рассчитать стоимость ремонта автомобиля:
- Выбор автомобиля (марка, модель, год, двигатель)
- Выбор категории ремонта (ТО, тормоза, подвеска и т.д.)
- Подбор запчастей с интеграцией с B2B-сервисом **BM Parts** (`https://developer.bm.parts/api/v2/`)
- Отображение цен и наличия в реальном времени
- Итоговый расчёт (запчасти + работа)

## Технологический стек

| Компонент | Технология |
|-----------|-----------|
| Backend | PHP 8.1+, WordPress 6.x |
| API Integration | REST API (WP) + HTTP-клиент к BM Parts API v2 |
| Frontend | Vanilla JS / Alpine.js (для калькулятора) |
| Кэширование | WordPress Transients API |

## Структура плагина

```
repair-calculator/
├── repair-calculator.php          # Главный файл плагина
├── includes/
│   ├── class-bm-parts-api.php    # HTTP-клиент для BM Parts API
│   ├── class-calculator.php      # Логика калькулятора
│   ├── class-admin-settings.php  # Настройки плагина
│   └── class-shortcode.php       # Shortcode [repair_calculator]
├── assets/
│   ├── js/
│   │   └── calculator.js         # Фронтенд-логика
│   └── css/
│       └── calculator.css        # Стили
├── templates/
│   └── calculator-form.php       # Шаблон формы
└── docs/                         # ← Эта документация
    ├── README.md
    ├── API_INTEGRATION.md
    └── endpoints/
        └── 01-search-part-by-name.md
```

## Документация

| Документ | Описание |
|----------|----------|
| [API_INTEGRATION.md](./API_INTEGRATION.md) | Общая интеграция с BM Parts API v2 |
| [01-search-part-by-name.md](./endpoints/01-search-part-by-name.md) | Эндпоинт поиска запчасти по названию |

## Этапы разработки

1. **Этап 1** — Интеграция с BM Parts API (поиск запчастей)
2. **Этап 2** — Форма калькулятора (выбор авто, категории)
3. **Этап 3** — Логика расчёта стоимости
4. **Этап 4** — Админ-панель (настройки API-ключа, маржа)
5. **Этап 5** — Кэширование и оптимизация

---

> ⚠️ **Важно:** Документация эндпоинтов BM Parts API основана на описании портала `developer.bm.parts`. Точные контракты (поля запроса/ответа) необходимо уточнить после получения доступа к API-документации.
