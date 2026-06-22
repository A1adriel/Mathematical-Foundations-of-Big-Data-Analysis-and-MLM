# Отчёт по лабораторной работе
## «Model Interpretability and Explainability»

**Выполнил:** Самбуев Алдар Баирович
**Дата:** 2026-06-20

---

## 1. Входные данные и предпосылки

- Feature set для `medical`: **robust_D** (выбран на основе ЛР-01, включает `num__cholesterol`, `num__glucose`, `num__systolic_bp`, `num__diastolic_bp`, `num__weight` и др.)
- Feature set для `finance`: **shortlist** (топ-12 признаков по фильтрам)

**Обоснование выбора:** robust_D даёт метрики, близкие к полному набору, но проще для интерпретации; shortlist удаляет шумовые признаки и улучшает стабильность коэффициентов.

---

## 2. Глоссарий

Файл: `study-notes/glossary.md` — добавлено 7 новых терминов.

| Термин | Практическая роль в работе |
|--------|---------------------------|
| глобальная интерпретация | общая оценка важности признаков по всей модели |
| native importance | базовая важность из дерева решений, смещена к числовым признакам |
| permutation importance | скорректировала смещение RandomForest |
| partial dependence plot | выявил нелинейный порог по возрасту |
| perturbation analysis | объяснил ошибки модели на конкретных объектах |
| local explanation | интерпретация предсказания для одного объекта |
| counterfactual | минимальное изменение признаков, меняющее предсказание |

---

## 3. Глобальная интерпретация моделей

| Dataset | Model | Method | Топ-5 признаков | Комментарий |
|---------|-------|--------|----------------|-------------|
| medical | LogisticRegression | Coef abs | cholesterol, glucose, systolic_bp, diastolic_bp, weight | Физиологические показатели — главные факторы риска |
| medical | LogisticRegression | Permutation | age, smoking, bmi, cholesterol, glucose | Permutation поднимает возраст и курение выше |
| medical | RandomForest | Native importance | bmi, age, cholesterol, smoking, exercise | Числовые признаки доминируют |
| medical | RandomForest | Permutation | age, smoking, bmi, cholesterol, glucose | Permutation снижает важность bmi |
| finance | LogisticRegression | Coef abs | credit_score, income, debt_ratio, previous_default, employment_years | Кредитный скоринг — самый сильный предиктор |
| finance | LogisticRegression | Permutation | credit_score, income, previous_default, debt_ratio, age | previous_default становится важнее |
| finance | RandomForest | Native importance | credit_score, income, previous_default, debt_ratio, loan_amount | Схоже с LR |
| finance | RandomForest | Permutation | credit_score, income, previous_default, debt_ratio, employment_years | Почти то же самое |

**Выводы:**
- Permutation importance корректирует смещение RandomForest в сторону признаков с большим числом уникальных значений (bmi, age).
- Коэффициенты логистической регрессии легко интерпретировать, но чувствительны к масштабированию и кодированию категорий.

---

## 4. Partial Dependence и устойчивость интерпретации

| Dataset | Model | Признак | Тренд | Score delta | Интерпретация |
|---------|-------|---------|-------|-------------|---------------|
| medical | RandomForest | age | increasing | ~0.32 | Риск растёт после 50 лет, резко после 60 |
| finance | LogisticRegression | credit_score | decreasing | ~0.45 | Чем выше скоринг, тем ниже риск дефолта (линейно) |

**Выводы:**
- PDP показывает, что возраст влияет нелинейно (порог ~50 лет).
- PDP может вводить в заблуждение на разреженных областях (например, возраст > 80).

---

## 5. Локальный разбор ошибок

- Самые уверенные ошибки: FP имеют score > 0.85, FN — score < 0.15.
- **Medical:** для FP наиболее значимы `family_history` и `age`; для FN — `bmi` и `smoking`.
- **Finance:** FP объясняются завышенным `debt_ratio` при среднем кредитном скоринге; FN — низким `credit_score` несмотря на хороший доход.

**Вывод:** perturbation-анализ показывает, что для разных типов ошибок важны разные признаки. Сегментный анализ по возрасту и доходу дополнил бы картину.

---

## 6. Практические рекомендации

- **Medical:** RandomForest на robust_D — выявляет нелинейные пороги (BMI, возраст) и даёт высокую точность.
- **Finance:** LogisticRegression на shortlist — простая интерпретация коэффициентов при почти тех же метриках, что у RandomForest.
- Если важна объяснимость, линейная модель предпочтительнее даже при небольшом проигрыше в качестве.

---

## 7. Самостоятельные задания

### 7.1. Согласованность глобальных объяснений

- Native importance и permutation importance согласованы в топ-3 для finance, но расходятся для medical (bmi vs smoking).
- Стабильные признаки: `age`, `smoking`, `credit_score`, `income`.
- Файл `outputs/global_importance_comparison.csv` создан.

### 7.2. Сводка partial dependence

- Наибольший `score_delta` у `credit_score` (0.45) — очень сильное влияние.
- `age` даёт монотонный рост, `bmi` — пороговый эффект после 30.
- Файл `outputs/partial_dependence_summary.csv` создан.

### 7.3. Локальные объяснения ошибок

- Medical FP: `family_history` и `age` — главные драйверы ошибки.
- Finance FN: `credit_score` и `debt_ratio` наиболее важны.
- Файл `outputs/error_case_explanations.csv` создан.

---

## 8. Проверка понимания

1. **Коэффициенты логистической регрессии нельзя интерпретировать без учёта препроцессинга** — масштабирование меняет величину коэффициентов, а one-hot кодирование создаёт базовый уровень.
2. **Permutation importance** может расходиться с native, потому что native переоценивает коррелированные признаки и признаки с большим числом разрывов (например, age), а permutation лишён этого смещения.
3. **Partial dependence** стоит трактовать осторожно на разреженных областях или при сильных взаимодействиях — усреднение может скрыть важные подгруппы.
4. **Локальное объяснение ошибки** не равно причинному объяснению — оно основано на корреляциях в данных, а не на контролируемом эксперименте; модель может использовать нерелевантные признаки-посредники.

---

## 9. Возможные улучшения

- Добавить анализ устойчивости объяснений на разных `random_state` (bootstrap).
- Сравнить с SHAP для более детальной локальной интерпретации.
- Провести сегментный анализ ошибок по возрасту и кредитному рейтингу.
- Визуализировать PD-кривые с доверительными интервалами.
