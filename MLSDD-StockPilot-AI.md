# MLSDD — StockPilot AI

- **Документ:** Machine Learning System Design Document
- **Версия:** 1.0
- **Дата:** 2026-10-01
- **Статус:** Draft for implementation

## 1. Executive summary

StockPilot AI — decision-support система для малого/среднего retail, которая принимает историю продаж, остатки, поставки и lead time, строит прогноз спроса и превращает его в объяснимую рекомендацию по закупке.

Цель MVP — доказать не «наличие AI», а улучшение решения по запасам относительно простых baseline на историческом backtest и пилоте.

Система **не** выполняет заказ автоматически. Пользователь подтверждает решение.

---

## 2. Problem statement

### 2.1. Business problem

Закупщик балансирует между двумя потерями:

- stockout → lost sales/service degradation;
- overstock → замороженный капитал, хранение, списания.

Текущие решения часто используют опыт, средние продажи и ручной Excel.

### 2.2. ML problem

Для каждой пары `sku × location` прогнозировать распределение/точечный спрос на горизонт, достаточный для покрытия `lead_time + review_period`, и затем использовать прогноз в replenishment policy.

### 2.3. Пользовательское решение

Для каждого SKU:

- `order_now?`;
- `recommended_qty`;
- `risk_stockout`;
- `overstock_flag`;
- `confidence`;
- `reason_codes`.

---

## 3. Goals and non-goals

### Goals

1. Поддержать CSV/XLSX импорт.
2. Обнаружить проблемы качества данных.
3. Построить baseline и candidate forecast.
4. Выполнить time-series backtesting.
5. Рассчитать закупочную рекомендацию.
6. Показать объяснение из проверяемых числовых факторов.
7. Экспортировать список действий.
8. Логировать версию данных, модели и рекомендации.

### Non-goals MVP

- автоматическая отправка PO поставщику;
- замена ERP/1С;
- dynamic pricing;
- multi-echelon optimization;
- RL/autonomous purchasing;
- прогноз макроэкономики;
- fully-online model serving.

---

## 4. Stakeholders

| Роль | Интерес |
|---|---|
| Владелец/операционный директор | капитал в запасах, продажи, ROI |
| Руководитель закупок | качество и скорость решений |
| Закупщик | понятная очередь действий |
| ML engineer | качество forecast/policy |
| Backend engineer | надежность pipeline/API |
| Data owner клиента | безопасность и качество данных |

---

## 5. Primary use cases

### UC-1. Ежедневная/еженедельная закупка

Пользователь открывает приоритетный список SKU и видит recommended quantity.

### UC-2. Разбор риска дефицита

Пользователь фильтрует SKU с высоким stockout risk до следующей поставки.

### UC-3. Поиск лишнего запаса

Пользователь видит SKU, запас которых сильно превышает ожидаемый спрос.

### UC-4. What changed?

Для SKU показывается изменение прогноза/риска относительно прошлого запуска и причины.

---

## 6. Data model

### 6.1. Core entities

#### `sales`

- `date`
- `location_id`
- `sku_id`
- `units_sold`
- `revenue` optional
- `price` optional
- `promo_flag` optional

#### `inventory_snapshot`

- `timestamp/date`
- `location_id`
- `sku_id`
- `on_hand`

#### `receipts`

- `date`
- `location_id`
- `sku_id`
- `qty_received`
- `supplier_id`

#### `purchase_orders`

- `po_id`
- `order_date`
- `expected_date`
- `sku_id`
- `qty_ordered`
- `qty_received`

#### `sku_master`

- `sku_id`
- `category`
- `brand`
- `unit_cost` optional
- `shelf_life` optional

#### `supplier`

- `supplier_id`
- `lead_time_days`
- `min_order_qty` optional
- `order_multiple` optional

---

## 7. Data contract and validation

### Hard validation

- даты парсятся;
- `sku_id` не пуст;
- `units_sold >= 0` после отдельной обработки возвратов;
- дубликаты определяются по ключу;
- единицы измерения согласованы.

### Soft validation

- >= 180 дней истории для ML режима;
- >= 80% expected daily coverage;
- lead time присутствует минимум на уровне supplier/category;
- аномальные spikes помечаются, но не удаляются молча;
- отрицательные остатки считаются data issue.

### Data quality score

`DQ = weighted(completeness, consistency, history_length, inventory_coverage, lead_time_coverage)`

Пороговая логика:

- `DQ >= 0.8` → ML allowed;
- `0.6 <= DQ < 0.8` → ML + warning;
- `< 0.6` → rules_only/insufficient_data.

Точные веса настраиваются после пилота.

---

## 8. Target definition

### MVP target

`y[s,l,t] = observed units sold` при условии достаточной доступности товара.

### Censored target

Если `on_hand == 0` часть дня/периода, наблюдаемая продажа может быть censored. Такие точки нельзя автоматически считать нулевым спросом.

MVP policy:

1. определить stockout-flag;
2. исключить явно censored intervals из некоторых feature calculations или импутировать conservative estimate;
3. хранить `target_quality`;
4. проводить отдельный error analysis на censored rows.

---

## 9. Feature engineering

### Time features

- day_of_week;
- week_of_year;
- month;
- weekend;
- holiday;
- days_to_holiday.

### Lag features

- lag 1, 2, 7, 14, 28;
- rolling mean/median 7/14/28;
- rolling std;
- rolling nonzero rate.

### Price/promo

- current price;
- price change %;
- promo flag;
- days since promo start.

### Inventory

- on_hand;
- days_of_supply baseline;
- stockout flag;
- recent stockout rate.

### Static

- category;
- brand;
- location;
- supplier/lead-time bucket.

Все rolling/lag features строятся только из прошлого относительно forecast origin.

---

## 10. Model strategy

### Baseline 0: seasonal naive

Основной контроль. Для daily retail часто разумная первая гипотеза — значение/среднее соответствующего дня недели прошлой недели.

### Baseline 1: moving average / ETS

Для стабильных fast-moving SKU.

### Candidate 1: global gradient boosted trees

LightGBM/CatBoost на lag/calendar/price features.

Преимущества:

- один global model использует информацию между SKU;
- быстрый train/inference;
- работает с нелинейностями;
- удобен для feature importance;
- проще эксплуатации deep model.

### Candidate 2 — post-MVP

Probabilistic model/quantile LightGBM или специализированная time-series модель для оценки uncertainty.

### Model routing

- insufficient history → rule/moving average;
- intermittent demand → intermittent baseline;
- normal series → global model;
- model failure → seasonal naive.

---

## 11. Training pipeline

1. Dataset snapshot immutable.
2. Schema validation.
3. Time normalization.
4. Stockout/censor flags.
5. Feature generation.
6. Temporal split.
7. Baseline evaluation.
8. Candidate tuning on validation windows.
9. Final evaluation on untouched holdout.
10. Model registry with metadata.

### Reproducibility metadata

- git commit;
- dataset version/hash;
- feature version;
- model params;
- train window;
- validation windows;
- metrics by segment.

---

## 12. Validation design

### Rolling-origin example

- Fold A: train months 1–8 → validate month 9
- Fold B: train 1–9 → validate month 10
- Fold C: train 1–10 → validate month 11
- Holdout: month 12

Горизонт зависит от lead time, но минимум отдельно оцениваются 7/14/28 дней.

### Segment evaluation

Обязательно разбивать результаты:

- ABC по revenue/margin;
- XYZ по variability;
- fast/intermittent;
- категории;
- locations;
- short/long lead time.

Средняя метрика не должна скрывать провал критичных SKU.

---

## 13. Metrics

### Forecast quality

- MASE;
- WAPE;
- bias;
- optional RMSSE для M5-like benchmark.

### Business proxy

- stockout rate;
- fill-rate proxy;
- average inventory;
- inventory turns;
- simulated lost margin;
- simulated holding cost;
- total policy cost.

### Product metrics

- recommendation acceptance rate;
- edit rate;
- time-to-order-list;
- percentage of SKU with usable forecast;
- percentage of low-confidence recommendations.

---

## 14. Acceptance criteria for MVP

Не фиксируем «магическое» требование к точности до первого benchmark. Предварительный gate:

1. Candidate median MASE лучше seasonal naive минимум на 10% в целевом сегменте **или** дает статистически/практически значимое улучшение business simulation.
2. WAPE не ухудшается на A-class SKU.
3. Bias находится в заранее согласованном диапазоне.
4. Replenishment simulation снижает total policy cost не менее чем на 5% против простого baseline на holdout **как исследовательский порог, не обещание клиенту**.
5. 95% рекомендаций содержат все расчетные компоненты и reason codes.
6. При low DQ система abstains/falls back, а не выдает ложную уверенность.

Порог 10%/5% подлежит пересмотру после первых benchmark и фиксируется до финального holdout.

---

## 15. Replenishment engine

### Inputs

- forecast horizon distribution/point forecast;
- on_hand;
- on_order;
- backorders optional;
- lead_time;
- review_period;
- MOQ/order multiple;
- service policy.

### Core calculation

`inventory_position = on_hand + on_order - backorders`

`protection_period = lead_time + review_period`

`expected_demand = sum(forecast over protection_period)`

`target_stock = expected_demand + safety_stock`

`raw_order = max(0, target_stock - inventory_position)`

`recommended_order = round_to_constraints(raw_order, MOQ, order_multiple)`

### Guardrails

- no negative orders;
- cap extreme jump unless user confirms;
- flag if lead_time missing;
- flag if recent stockout makes demand uncertain;
- display impact of MOQ.

---

## 16. Explanation layer

Не использовать LLM для вычисления quantity.

Сначала deterministic reason object:

```json
{
  "on_hand": 17,
  "on_order": 0,
  "forecast_protection_period": 39,
  "safety_stock": 8,
  "target_stock": 47,
  "recommended_order": 30,
  "trend_4w_pct": 16,
  "confidence": "medium"
}
```

Затем шаблон или LLM превращает объект в естественный язык. Любое число в тексте должно быть взято из объекта.

---

## 17. Serving architecture

### Batch first

Для MVP forecast запускается ночью или по кнопке после загрузки данных.

Компоненты:

- Web UI;
- FastAPI;
- PostgreSQL;
- object storage/Parquet;
- worker queue;
- feature/forecast job;
- model registry;
- recommendation service.

### Why batch

Закупочные решения обычно не требуют миллисекундной задержки; batch дешевле и проще, облегчает reproducibility и rollback.

---

## 18. API sketch

### POST `/datasets`

Загрузка файла, возвращает `dataset_id`.

### GET `/datasets/{id}/quality`

Data-quality report.

### POST `/forecast-runs`

Запуск batch forecast для dataset/model version.

### GET `/forecast-runs/{id}/recommendations`

Список рекомендаций.

### GET `/sku/{sku_id}/explanation`

Forecast, components, reason codes, historical chart.

### POST `/recommendations/{id}/feedback`

`accepted | edited | rejected`, optional reason.

---

## 19. Observability

### Data monitoring

- row counts;
- missingness;
- SKU coverage;
- feature distributions;
- stockout rate;
- lead-time coverage.

### Model monitoring

После появления факта:

- MASE/WAPE by week;
- bias;
- error by ABC/XYZ;
- prediction interval coverage if available.

### Product monitoring

- batch duration;
- failed jobs;
- fallback rate;
- low-confidence rate;
- recommendation acceptance/edit rate.

---

## 20. Drift and retraining

MVP policy:

- retrain weekly/monthly or after new upload;
- compare challenger vs current champion on recent rolling window;
- promote only if gates passed;
- retain previous model for rollback.

Drift signals:

- PSI/feature shifts optional;
- sustained forecast bias;
- rapid changes in price/promo mix;
- new SKU share.

---

## 21. New SKU / cold start

Если SKU новый:

- use category/store priors;
- simple rule based on analogous SKU if approved;
- show low confidence;
- do not fabricate long-term seasonality.

MVP can exclude new SKU < N days from ML and use rule fallback.

---

## 22. Security and privacy

- tenant isolation;
- encryption in transit/at rest;
- role-based access;
- no client data in public training sets;
- configurable retention;
- audit log for uploads and forecast runs;
- minimize PII: product/transaction data generally does not require customer-level personal data for this use case.

---

## 23. Failure modes

| Failure | Detection | Response |
|---|---|---|
| Sales gap | missingness check | exclude/flag period |
| Hidden stockout | inventory evidence | censored flag, lower confidence |
| Missing lead time | schema/DQ | no auto quantity, require input |
| Promo spike | feature/anomaly | reason code; avoid naive carry-forward |
| New SKU | history length | cold-start fallback |
| Model artifact missing | health check | seasonal naive |
| Extreme recommendation | guardrail | cap + manual review |
| Bad units | schema/business rule | block dataset |

---

## 24. Testing strategy

### Unit tests

- lag features contain no future values;
- reorder math;
- MOQ rounding;
- missing lead time behavior;
- explanation numbers equal source object.

### Data tests

- uniqueness;
- ranges;
- date continuity;
- referential integrity.

### ML tests

- baseline always runs;
- no train/test overlap;
- deterministic seed where possible;
- metric calculation fixtures;
- segment report generated.

### End-to-end

Golden dataset → upload → quality → forecast → recommendation → export.

---

## 25. Rollout plan

### Phase 0 — benchmark

M5/FreshRetailNet subset, reproducible notebook/pipeline.

### Phase 1 — internal MVP

CSV upload + dashboard + baseline + boosted model.

### Phase 2 — shadow pilot

20–100 SKU клиента, recommendations not used automatically.

### Phase 3 — assisted use

User reviews recommendation and records decision.

### Phase 4 — integration

Only after proof: 1С/MойСклад connector.

---

## 26. Experiment plan

### E1. Baseline vs global GBDT

Hypothesis: global GBDT improves MASE/WAPE for stable and seasonal SKU segments.

### E2. Forecast improvement vs inventory outcome

Hypothesis: lower forecast error improves simulated policy cost under fixed replenishment rule.

### E3. Censor-aware treatment

Hypothesis: flagging/imputing censored stockout periods reduces negative bias vs treating observed zero/low sales as demand.

### E4. Human usability

Hypothesis: prioritized list reduces median time to create order draft by >=30% in controlled task test.

---

## 27. Open design questions

- point vs probabilistic forecast in v1;
- whether store×SKU is always the atomic series;
- how to estimate safety stock for intermittent demand;
- whether margin/cost data are available;
- how to normalize supplier lead time volatility;
- how to handle substitutions/cannibalization;
- whether promotions are known in advance.

---

## 28. References

1. Makridakis et al., M5 Accuracy Competition. https://doi.org/10.1016/j.ijforecast.2021.11.013
2. Hyndman & Koehler, Another look at measures of forecast accuracy. https://doi.org/10.1016/j.ijforecast.2006.03.001
3. Trapero et al., Demand forecasting under lost sales stock policies. https://doi.org/10.1016/j.ijforecast.2023.09.004
4. FreshRetailNet-50K. https://arxiv.org/abs/2505.16319
5. Netstock 2026 Benchmark. https://www.netstock.com/research/supply-chain-planning-report/
6. 1С-Товары. https://portal.1c.ru/applications/1C-Goods
