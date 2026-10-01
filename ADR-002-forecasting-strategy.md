# ADR-002: Глобальная GBDT-модель с обязательным seasonal-naive fallback

- **Статус:** Accepted
- **Дата:** 2026-10-01

## Контекст

Нужно выбрать первую ML-стратегию для тысяч SKU при ограниченном времени проекта. Варианты: отдельная статистическая модель на каждый ряд, global gradient boosting, deep learning time-series model.

## Решение

Использовать:

1. **seasonal naive** как обязательный production baseline/fallback;
2. **global LightGBM/CatBoost** как основной candidate;
3. простые статистические/intermittent модели как segment-specific fallback.

Deep learning оставить вне MVP.

## Причины

- M5 показал конкурентоспособность global ML-подходов в retail forecasting.
- GBDT естественно использует lag, calendar, price/promo и категориальные признаки.
- Быстро обучается и проще отлаживается.
- Один global model легче поддерживать, чем тысячи индивидуально настроенных моделей.
- Seasonal naive дает надежный контроль и аварийный режим.

## Альтернативы

### Deep forecasting model

Плюсы: потенциально лучше захватывает сложные зависимости и uncertainty.  
Минусы: больше инфраструктуры, сложнее explain/debug, выше риск потратить учебный проект на tuning.

### Только статистические модели

Плюсы: простота и интерпретируемость.  
Минусы: хуже масштабирование на covariates/большой набор series; сложнее использовать общие закономерности между SKU.

## Последствия

- Feature pipeline становится критичным компонентом.
- Нужна строгая защита от time leakage.
- Improvement измеряется относительно seasonal naive, а не относительно «нулевой модели».
- Если GBDT не проходит gate, продукт остается работоспособным на baseline.

## Пересмотр

Пересмотреть после первого пилота, если:

- GBDT не дает стабильного улучшения;
- требуется probabilistic forecasting;
- data volume и business value оправдывают deep model.
