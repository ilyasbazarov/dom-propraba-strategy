# АППЕНДИКС B — СХЕМА ДАННЫХ И ЛОГИКА РАСЧЁТОВ
## Программа «Каталог 2.0» | Фаза 0

**Версия:** v1.0 | **Дата:** 2026-06-11

---

## B.1 НОВЫЕ КОЛОНКИ ДЛЯ ШАГА 0

Список полей, которые нужно добавить в снэпшот перед выгрузкой.
Все источники уже есть в существующем пайплайне — новых данных не требуется.

| Колонка | Тип | Источник | Описание |
|---|---|---|---|
| `in_stock_days_30` | int | леджер → агрегат | Кол-во дней is_in_stock_day=1 за 30д |
| `in_stock_days_90` | int | леджер → агрегат | Кол-во дней is_in_stock_day=1 за 90д |
| `adt_confidence` | str | расчёт | 'OK' или 'LOW' по правилам из B.2 |
| `target_stock_qty` | float | расчёт | ADT_90 × Target_Coverage_Days |
| `excess_qty` | float | расчёт | max(0, Current_Stock_Qty − target_stock_qty) |
| `working_qty` | float | расчёт | min(Current_Stock_Qty, target_stock_qty) |
| `working_capital` | float | расчёт | working_qty × Current_Cost |
| `excess_capital` | float | расчёт | excess_qty × Current_Cost |
| `effective_coverage_days` | float | расчёт | Current_Stock_Qty / True_ADT_90 (inf если ADT=0) |
| `stock_health` | str | расчёт | HEALTHY / OVERSTOCK / EXCESS / DEAD |
| `sku_age_days` | int | леджер | Дней с первой транзакции по баркоду |
| `is_seasonal` | int | ручной список | 1 если категория сезонная (список в A.5) |
| `quarantine_flag` | int | расчёт | 0 = чистый, 1 = JUNK, 2 = FIX |
| `quarantine_type` | str | расчёт | '' / 'QUARANTINE_JUNK' / 'QUARANTINE_FIX' |
| `role_auto` | str | расчёт | LOCOMOTIVE / MARGIN_ENGINE / TEST_POOL / KILL_CANDIDATE / COMPLETER |
| `role_override` | str | ручной | Ручной override при необходимости, иначе '' |
| `assortment_layer` | str | расчёт | = role_override если заполнен, иначе role_auto |

**Итого: 17 новых колонок.** Все вычисляются поверх существующих полей.

---

## B.2 ЛОГИКА РАСЧЁТА НОВЫХ КОЛОНОК (псевдокод)

### in_stock_days_30, in_stock_days_90

```python
# Из леджера, агрегат по баркоду
in_stock_days_30 = ledger.filter(
    date >= today - 30,
    is_in_stock_day == 1
).groupby('Barcode').count()

in_stock_days_90 = ledger.filter(
    date >= today - 90,
    is_in_stock_day == 1
).groupby('Barcode').count()
```

---

### adt_confidence

```python
def calc_adt_confidence(in_stock_days_30, in_stock_days_90):
    if in_stock_days_90 < 15:
        return 'LOW'
    if in_stock_days_30 < 7 and in_stock_days_90 < 20:
        return 'LOW'
    return 'OK'
```

**Важно:** True_ADT_90 уже должен быть рассчитан через in_stock_days (не calendar_days).
Если в текущем пайплайне True_ADT_90 = Sold_Qty_90D / 90 — нужно пересчитать:
```python
True_ADT_90_corrected = Sold_Qty_90D / max(in_stock_days_90, 1)
# Затем winsorize по P95 внутри Category_L2
```

---

### target_stock_qty, excess_qty, working_qty

```python
TARGET_COVERAGE_DAYS = 60  # дефолт (параметр A.3)
EPSILON = 0.033            # мин. значимый ADT (параметр A.3)

def calc_stock_decomposition(row):
    adt = row['True_ADT_90']  # уже скорректированный
    
    # Если ADT ненадёжен или ниже epsilon
    if row['adt_confidence'] == 'LOW' or adt < EPSILON:
        target = 0.0
    else:
        target = adt * TARGET_COVERAGE_DAYS
    
    current = row['Current_Stock_Qty']
    excess = max(0.0, current - target)
    working = min(current, target)
    
    return target, excess, working
```

---

### working_capital, excess_capital

```python
working_capital = working_qty * Current_Cost
excess_capital  = excess_qty  * Current_Cost
```

---

### effective_coverage_days

```python
def calc_coverage(row):
    adt = row['True_ADT_90']
    if adt < EPSILON:
        return float('inf')  # в отчётах показывать как 9999 или '∞'
    return row['Current_Stock_Qty'] / adt
```

---

### stock_health

```python
DEAD_COVERAGE_THRESHOLD    = 365   # дней (параметр A.3)
EXCESS_COVERAGE_THRESHOLD  = 180
OVERSTOCK_COVERAGE_THRESHOLD = 90
DEAD_LKP_AGE_THRESHOLD     = 180   # дней (параметр A.3)
DEAD_STAGNATION_THRESHOLD  = 90    # дней (параметр A.3)

def calc_stock_health(row):
    adt    = row['True_ADT_90']
    cov    = row['effective_coverage_days']
    lkp    = row['LKP_Age_Days']
    stag   = row['Current_Stagnation_Days']
    stock  = row['Current_Stock_Qty']
    
    # Предохранитель: новинки → не классифицировать
    if row['sku_age_days'] <= 120:
        return 'NEW_SKU'
    
    # Сезонные → не классифицировать стандартно
    if row['is_seasonal'] == 1:
        return 'SEASONAL_HOLD'
    
    # DEAD: требует двух независимых сигналов
    signal_1 = (adt < EPSILON and stock > 0) or (cov > DEAD_COVERAGE_THRESHOLD)
    signal_2 = (lkp > DEAD_LKP_AGE_THRESHOLD) or (stag > DEAD_STAGNATION_THRESHOLD)
    
    if signal_1 and signal_2:
        return 'DEAD'
    
    # EXCESS: один сигнал
    if signal_1:
        return 'EXCESS'
    
    if cov > EXCESS_COVERAGE_THRESHOLD:
        return 'EXCESS'
    
    if cov > OVERSTOCK_COVERAGE_THRESHOLD:
        return 'OVERSTOCK'
    
    return 'HEALTHY'
```

---

### sku_age_days

```python
# Из леджера: первая транзакция по баркоду
first_transaction = ledger.groupby('Barcode')['Date'].min()
sku_age_days = (today - first_transaction).days
```

---

### quarantine_flag, quarantine_type

```python
def calc_quarantine(row):
    # JUNK: мусор без живых данных
    is_junk = (
        str(row['Barcode']).startswith('INT-')
        and row['Primary_Supplier'] == 'UNKNOWN_SUPPLIER'
        and row['Category_L1'] == '00 БЕЗ КАТЕГОРИИ'
        and row['Revenue_90D'] == 0
    )
    if is_junk:
        return 1, 'QUARANTINE_JUNK'
    
    # FIX: живой товар с плохими мастер-данными
    is_fix = (
        (row['Category_L1'] == '00 БЕЗ КАТЕГОРИИ' and row['Revenue_90D'] > 0)
        or (row['Primary_Supplier'] == 'UNKNOWN_SUPPLIER' 
            and row['ABC_Revenue_90D'] == 'A')
        or (str(row['Barcode']).startswith('INT-') 
            and row['Revenue_90D'] > 0)
    )
    if is_fix:
        return 2, 'QUARANTINE_FIX'
    
    return 0, ''
```

---

### role_auto

```python
def calc_role_auto(row, locomotive_threshold, turnover_threshold, median_turnover):
    # Карантин → не назначать роль
    if row['quarantine_flag'] > 0:
        return 'QUARANTINE'
    
    # Новинки → тестовый пул
    if row['sku_age_days'] <= 120:
        return 'TEST_POOL'
    
    # Мертвецы → кандидаты на kill
    if row['stock_health'] in ('DEAD', 'EXCESS'):
        # Но только если excess_capital > materiality threshold
        if row['excess_capital'] >= 3000:
            return 'KILL_CANDIDATE'
    
    # Локомотивы
    monthly_revenue = row['Revenue_90D'] / 3  # апроксимация мес выручки
    turnover_q = row['GMROI_Annualized_90D'] / 4 if row['GMROI_Annualized_90D'] else 0
    # Примечание: используй реальный turnover_q если есть, 
    # иначе апроксимируй через GMROI или Cost_Sales / Avg_Stock
    
    if monthly_revenue >= locomotive_threshold and turnover_q >= turnover_threshold:
        return 'LOCOMOTIVE'
    
    # Маржинальные движки
    margin_rate = row['Unit_Margin_T0'] / row['Current_Retail_Price'] if row['Current_Retail_Price'] > 0 else 0
    if margin_rate >= 0.40 and row['True_ADT_90'] >= median_turnover:
        return 'MARGIN_ENGINE'
    
    # Все остальные
    return 'COMPLETER'
```

**Примечание:** `locomotive_threshold` и `turnover_threshold` — финализируются после Шага 2 (расчёт порогов). До этого role_auto для LOCOMOTIVE не считать, оставить как COMPLETER. После Шага 2 — пересчитать.

---

### assortment_layer

```python
assortment_layer = role_override if role_override != '' else role_auto
```

---

## B.3 СХЕМА СУЩЕСТВУЮЩЕЙ ВИТРИНЫ (снэпшот)

Все поля существующего снэпшота с типами и кратким описанием.

| Колонка | Тип | Описание |
|---|---|---|
| `Barcode` | str | Основной идентификатор SKU |
| `Item_Name` | str | Наименование товара |
| `Category_L1` | str | Категория уровень 1 |
| `Category_L2` | str | Категория уровень 2 |
| `Category_L3` | str | Категория уровень 3 |
| `Current_Retail_Price` | float | Текущая розничная цена |
| `Current_Cost` | float | Текущая себестоимость |
| `Unit_Margin_T0` | float | Абсолютная маржа на единицу |
| `LKP_Age_Days` | int | Дней с последней известной закупки (last known price) |
| `Current_Stock_Qty` | float | Остаток на складе |
| `Current_Frozen_Capital` | float | Замороженный капитал = Current_Stock × Cost |
| `Coverage_Days_30` | float | Дней покрытия по ADT 30д |
| `True_ADT_30` | float | Скорректированный ADT за 30д |
| `True_ADT_90` | float | Скорректированный ADT за 90д |
| `Sold_Qty_30D` | float | Продано штук за 30д |
| `Sold_Qty_90D` | float | Продано штук за 90д |
| `Revenue_30D` | float | Выручка за 30д |
| `Revenue_90D` | float | Выручка за 90д |
| `Gross_Profit_30D` | float | Валовая прибыль за 30д |
| `Gross_Profit_90D` | float | Валовая прибыль за 90д |
| `GMROI_Annualized_90D` | float | Аннуализированный GMROI по 90д |
| `Lost_Revenue_30D` | float | Потерянная выручка из-за OOS за 30д |
| `Lost_Revenue_90D` | float | Потерянная выручка из-за OOS за 90д |
| `Lost_GP_30D` | float | Потерянная GP из-за OOS за 30д |
| `Lost_GP_90D` | float | Потерянная GP из-за OOS за 90д |
| `ABC_Revenue_90D` | str | ABC по выручке 90д (A/B/C) |
| `ABC_Margin_90D` | str | ABC по марже 90д |
| `ABC_Cross_Class` | str | Кросс-класс ABC (AA/AB/BA/...) |
| `XYZ_Class_90D` | str | XYZ по стабильности (X/Y/Z) |
| `CoV_90D` | float | Коэффициент вариации спроса 90д |
| `is_KGT` | int | Крупногабаритный товар (1/0) |
| `Ecom_Status` | str | Статус для e-com |
| `Toxicity_Status` | str | Статус токсичности (старая модель) |
| `OOS_Action` | str | Рекомендуемое действие при OOS |
| `Current_Stagnation_Days` | int | Дней стагнации |
| `is_cash_trap` | int | Флаг «ловушка капитала» (старая логика) |
| `Primary_Supplier` | str | Основной поставщик |
| `All_Suppliers` | str | Все поставщики через разделитель |
| `Supplier_Count` | float | Количество поставщиков |
| `All_Suppliers_map` | str | Маппинг поставщиков |
| `Strategic_Quadrant` | str | Стратегический квадрант поставщика |
| `ABC_Supplier_Margin` | str | ABC поставщика по марже |
| `ABC_Supplier_Health` | str | Здоровье поставщика |

---

## B.4 СХЕМА ЛЕДЖЕРА (для справки)

Леджер используется точечно: для верификации подозрительных ADT и расчёта sku_age_days.
Полная выгрузка леджера не требуется на Шаге 1.

| Колонка | Тип | Описание |
|---|---|---|
| `Date` | date | Дата записи |
| `Barcode` | str | Идентификатор SKU |
| `Daily_Stock_Qty` | float | Остаток на конец дня |
| `Qty_Logistics` | float | Количество в логистике |
| `Revenue_Financial` | float | Выручка дня |
| `Cost_Financial` | float | Себестоимость продаж дня |
| `pure_inbound_qty` | float | Чистый приход |
| `return_to_supplier_qty` | float | Возврат поставщику |
| `daily_avg_purchase_price` | float | Средняя закупочная цена дня |
| `Cost_Per_Unit` | float | Себестоимость единицы |
| `Final_Cost_Per_Unit` | float | Финальная себестоимость |
| `Ledger_Frozen_Capital` | float | Замороженный капитал в леджере |
| `Business_Frozen_Capital` | float | Бизнес-значение замороженного капитала |
| `Daily_Margin_Absolute` | float | Абсолютная маржа дня |
| `is_cost_compromised` | int | Флаг ненадёжной себестоимости |
| `is_price_filled` | int | Флаг заполненной цены |
| `is_negative_stock_day` | int | Флаг отрицательного остатка |
| `is_phantom_sale` | int | Флаг фантомной продажи |
| `is_zero_movement_day` | int | Нет движения за день |
| `is_margin_negative` | int | Отрицательная маржа дня |
| `Item_Name` | str | Наименование |
| `Category_L1/L2/L3` | str | Категории |
| `is_in_stock_day` | int | **Ключевое поле:** 1 = товар был в наличии |
| `Rolling_Median_90d` | float | Скользящая медиана 90д |
| `Qty_Threshold_Limit` | float | Пороговое количество |
| `is_b2b_anomaly` | int | Флаг B2B-аномалии |
| `Qty_Logistics_Clean` | float | Очищенное количество в логистике |
| `True_ADT_30` | float | ADT 30д |
| `True_ADT_90` | float | ADT 90д |
| `Demand_StdDev_30` | float | Стандартное отклонение спроса 30д |
| `Target_Margin_Rate` | float | Целевая маржинальность |
| `Lost_Revenue_Daily` | float | Потерянная выручка за день |
| `Lost_Gross_Profit_Daily` | float | Потерянная GP за день |
| `Shelf_Stagnation_Days` | int | Дней стагнации на полке |
| `is_toxic_candidate_day` | int | Флаг кандидата в токсичные |

---

## B.5 ФЛАГИ КАЧЕСТВА ДАННЫХ

| Флаг | Источник | Что означает | Действие при анализе |
|---|---|---|---|
| `is_cost_compromised = 1` | леджер | Себестоимость ненадёжна (пересчёты, ошибки) | Использовать с осторожностью. Маржу по таким SKU не считать надёжной. |
| `is_phantom_sale = 1` | леджер | Продажа без реального движения товара | Исключать из ADT-расчётов |
| `is_b2b_anomaly = 1` | леджер | B2B-всплеск, нерепрезентативный для обычного спроса | Исключать из ADT для прогноза розничного спроса |
| `is_negative_stock_day = 1` | леджер | Отрицательный остаток (учётная ошибка) | Флаг при интерпретации GMROI и Coverage |
| `adt_confidence = 'LOW'` | новое поле | ADT статистически ненадёжен | Не использовать Lost_Revenue, Target_Stock, Coverage для принятия решений по этому SKU |
| `quarantine_flag > 0` | новое поле | Плохие мастер-данные | Исключать из основного анализа, репортить отдельно |
| `is_KGT = 1` | существующее | Крупногабаритный товар | Delivery cost correction при unit-экономике |
| `LKP_Age_Days > 180` | существующее | Устаревшая цена закупки | Маржа может быть устаревшей, один из сигналов DEAD |

---

## B.6 ПРОВЕРОЧНЫЕ СУММЫ (для гейта Шага 0)

После пересчёта снэпшота — проверить:

```python
# 1. Общий frozen capital (допустимое расхождение ±5%)
assert abs(df['Current_Frozen_Capital'].sum() - df['working_capital'].sum() 
           - df['excess_capital'].sum()) / df['Current_Frozen_Capital'].sum() < 0.05

# 2. Quarantine не потерял строки
assert len(df) == original_row_count  # строки маркируются, не удаляются

# 3. stock_health покрывает все строки кроме NEW_SKU и SEASONAL_HOLD
valid_health = ['HEALTHY', 'OVERSTOCK', 'EXCESS', 'DEAD', 'NEW_SKU', 'SEASONAL_HOLD']
assert df['stock_health'].isin(valid_health).all()

# 4. excess_capital >= 0 для всех строк
assert (df['excess_capital'] >= 0).all()

# 5. working_capital + excess_capital = Current_Frozen_Capital (±1 сом на строку — рондинг)
diff = abs(df['working_capital'] + df['excess_capital'] - df['Current_Frozen_Capital'])
assert (diff < 1.0).all()  # рублевая точность
```

---

*Конец Аппендикса B*
