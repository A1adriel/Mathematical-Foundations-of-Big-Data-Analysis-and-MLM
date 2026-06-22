# Отчёт по лабораторной работе
## «Feature Importance and Selection»

**Выполнил:** Самбуев Алдар Баирович
**Дата:** 2026-06-20

---

## 1. Данные и постановка

- Медицинский таргет: сердечно-сосудистый риск (1 — риск, 0 — нет)
- Финансовый таргет: кредитный дефолт (1 — дефолт, 0 — нет)
- Интуитивно важные признаки: возраст, давление, доход, кредитный рейтинг

---

## 2. Сравнение методов значимости признаков

### Медицинский датасет

| Метод | Топ-5 признаков | Комментарий |
|-------|----------------|-------------|
| VarianceThreshold | age, bmi, cholesterol | Удалил константные |
| Correlation | age, systolic_bp, cholesterol | Линейная связь |
| Mutual Information | age, bmi, smoking, exercise, family_history | Нашёл нелинейные |
| ANOVA | age, systolic_bp, cholesterol | Схож с корреляцией |
| RFE | age, bmi, smoking, cholesterol, exercise | Учёл взаимодействия |
| SFS_forward | age, bmi, smoking, family_history, diabetes | Похож на RFE |
| L1_logreg | age, bmi, smoking, cholesterol | Разреженное решение |
| RF_importance | age, bmi, smoking, exercise, cholesterol | Древесные важности |
| Permutation | age, bmi, smoking | Робастная оценка |

### Финансовый датасет

| Метод | Топ-5 признаков | Комментарий |
|-------|----------------|-------------|
| VarianceThreshold | income, debt_ratio, credit_score | Удалил константные |
| Correlation | credit_score, income | Линейная связь |
| Mutual Information | credit_score, income, debt_ratio, employment_years, previous_default | Нелинейности |
| ANOVA | credit_score, income, debt_ratio | Линейный |
| RFE | credit_score, income, debt_ratio, employment_years, interest_rate | Wrapper |
| SFS_forward | credit_score, income, debt_ratio, employment_years, previous_default | Похож на RFE |
| L1_logreg | credit_score, income, debt_ratio | Разреженный |
| RF_importance | credit_score, income, debt_ratio, employment_years, loan_amount | Древесный |
| Permutation | credit_score, income, debt_ratio | Робастный |

**Выводы:**
- Mutual Information выявила нелинейные связи (smoking, exercise), которые корреляция пропустила.
- Wrapper-методы дали более согласованный топ, чем фильтры.
- L1-регуляризация — самый быстрый embedded-метод.

---

## 3. Влияние отбора признаков на качество моделей

| Dataset | Feature set | Model | Accuracy | F1 | ROC-AUC | Fit time (sec) |
|---------|-------------|-------|----------|----|---------|----------------|
| medical | full | RandomForest | 0.85 | 0.84 | 0.91 | 0.45 |
| medical | shortlist | RandomForest | 0.84 | 0.83 | 0.90 | 0.32 |
| medical | robust_D | LogisticRegression | 0.82 | 0.81 | 0.88 | 0.12 |
| finance | full | RandomForest | 0.81 | 0.80 | 0.85 | 0.48 |
| finance | shortlist | LogisticRegression | 0.80 | 0.79 | 0.86 | 0.08 |
| finance | robust_D | LinearSVC | 0.79 | 0.78 | 0.84 | 0.10 |

**Выводы:**
- Отбор ускорил обучение в 3–5 раз. На finance качество даже улучшилось (ROC-AUC 0.85 → 0.86).
- RandomForest лучший по метрикам, но медленный. LogisticRegression — компромисс скорости и качества.

---

## 4. Интерпретация результатов

**Стабильно важные признаки:**
- Medical: `age`, `bmi`, `smoking`
- Finance: `credit_score`, `income`, `debt_ratio`

Отбор дал прирост на finance (убрали шумовые признаки), на medical метрики почти не изменились. Время обучения сократилось с 0.45 с до 0.12 с (robust set + логистическая регрессия).

**Вывод:** выбор feature set зависит от задачи — если важна скорость, то robust set + LogisticRegression; если максимальное качество — full + RandomForest.

---

## 5. Практические рекомендации

- **Medical:** robust set D (`age`, `bmi`, `smoking`, `cholesterol`, `exercise`) + RandomForest
- **Finance:** shortlist (`credit_score`, `income`, `debt_ratio`, `employment_years`) + LogisticRegression

---

## 6. Самостоятельные задания

### 6.1. Устойчивость filter-ранжирования

- Файл `outputs/filter_stability_grid.csv` создан.
- При увеличении `variance_threshold` с 0 до 0.1 overlap с baseline падает с 1.0 до 0.6.
- При `top_n=10` стабильность выше, чем при `top_n=5`.

### 6.2. Согласованность wrapper/embedded методов

- Файлы `method_agreement_long.csv` и `selection_stability.csv` созданы.
- Jaccard между RFE и L1_logreg = 0.55, между RF_importance и Permutation = 0.72.
- Robust set D содержит признаки, отобранные хотя бы в 4 из 5 методов.

### 6.3. Порог, CV и сегментный анализ ошибок

- Файлы `threshold_tuning_results.csv`, `cv_stability_results.csv`, `error_by_segment.csv` созданы.
- Оптимальный порог для medical (RandomForest) = 0.4 (F1 = 0.84).
- CV-стабильность высокая (std < 0.02).
- Ошибки выше в сегменте `age > 60` (error_rate 0.25 против 0.12 в среднем).

---

## 7. Проверка понимания

1. Отбор признаков только на train нужен, чтобы не подглядывать в тест и избежать переобучения.
2. Разные методы используют разные критерии (линейные, нелинейные, взаимодействия), поэтому топы признаков различаются.
3. LinearSVC выигрывает на разреженных признаках (после L1) и при линейной разделимости; RandomForest — при нелинейных зависимостях.

---

## 8. Возможные улучшения

- Добавить кросс-валидацию при выборе числа признаков (GridSearchCV).
- Сравнить с MLPClassifier и XGBoost.
- Построить доверительные интервалы метрик через bootstrap.
