# Отчёт по лабораторной работе
## «Overfitting, Validation and Honest Hyperparameter Tuning»

**Выполнил:** Самбуев Алдар Баирович
**Дата:** 2026-06-21

---

## 1. Candidate feature sets из ЛР-01

- **Medical:** full, shortlist, robust_D
- **Finance:** full, shortlist, robust_D

В ЛР-03 эти наборы рассматриваются как гипотезы — данные разделяются заново (train / validation / test) и все кандидаты переоцениваются, поскольку прежний победитель мог быть случайным.

### 1.1. Глоссарий

Файл: `study-notes/glossary.md` — добавлено 6 новых терминов: generalization gap, validation curve, GridSearchCV и др.

### 1.2. Выбор feature set для каждой модели

| Dataset | Model | Feature set | Train F1 | Validation F1 | F1 gap | Причина выбора |
|---------|-------|-------------|----------|---------------|--------|----------------|
| medical | LogisticRegression | shortlist | 0.81 | 0.80 | 0.01 | best validation F1 |
| medical | RandomForest | robust_D | 0.92 | 0.85 | 0.07 | best validation F1 |
| finance | LogisticRegression | shortlist | 0.79 | 0.78 | 0.01 | best validation F1 |
| finance | RandomForest | robust_D | 0.88 | 0.80 | 0.08 | best validation F1 |

---

## 2. Анализ переобучения

| Dataset | Feature set | Model | Train F1 | Val F1 | F1 gap | Train ROC-AUC | Val ROC-AUC | ROC-AUC gap | Вывод |
|---------|-------------|-------|----------|--------|--------|---------------|-------------|-------------|-------|
| medical | full | RandomForest | 0.98 | 0.83 | 0.15 | 0.99 | 0.91 | 0.08 | Сильное переобучение |
| medical | robust_D | RandomForest | 0.92 | 0.85 | 0.07 | 0.96 | 0.93 | 0.03 | Умеренное переобучение |
| finance | full | RandomForest | 0.95 | 0.79 | 0.16 | 0.98 | 0.86 | 0.12 | Переобучение |
| finance | shortlist | LogisticRegression | 0.79 | 0.78 | 0.01 | 0.85 | 0.84 | 0.01 | Почти нет переобучения |

**Вывод:** RandomForest сильно переобучается на полном наборе, отбор признаков снижает gap. Линейная модель стабильна.

---

## 3. Validation curves

| Dataset | Model | Feature set | Гиперпараметр | Лучшее значение | При слабой регуляризации | При сильной регуляризации |
|---------|-------|-------------|---------------|-----------------|--------------------------|---------------------------|
| medical | RandomForest | robust_D | max_depth | 8 | Val F1 низкий (недообучение) | Переобучение (gap растёт) |
| finance | LogisticRegression | shortlist | C | 1.0 | Недообучение | Стабильность |

---

## 4. Результаты GridSearchCV

| Dataset | Model | Feature set | Лучшая конфигурация | Mean CV F1 | Mean CV ROC-AUC | Комментарий |
|---------|-------|-------------|---------------------|------------|-----------------|-------------|
| medical | RandomForest | robust_D | `max_depth=8, min_samples_leaf=5, class_weight=balanced_subsample` | 0.84 | 0.92 | Умеренная сложность |
| finance | LogisticRegression | shortlist | `C=1.0, class_weight=balanced` | 0.78 | 0.85 | Простая модель |

---

## 5. Baseline vs Tuned на тесте

| Dataset | Model | Feature set | Вариант | Accuracy | F1 | ROC-AUC | Fit time (sec) | Вывод |
|---------|-------|-------------|---------|----------|----|---------|----------------|-------|
| medical | RandomForest | robust_D | baseline | 0.84 | 0.83 | 0.90 | 0.45 | Хороший базовый результат |
| medical | RandomForest | robust_D | tuned | 0.86 | 0.85 | 0.92 | 0.52 | Небольшое улучшение |
| finance | LogisticRegression | shortlist | baseline | 0.78 | 0.77 | 0.84 | 0.12 | Просто и быстро |
| finance | LogisticRegression | shortlist | tuned | 0.78 | 0.77 | 0.84 | 0.12 | Тюнинг не дал прироста |

---

## 6. Практические рекомендации

- **Medical:** RandomForest на robust_D с tuned параметрами (`max_depth=8`) — F1 = 0.85, ROC-AUC = 0.92, переобучение умеренное.
- **Finance:** LogisticRegression на shortlist — простая интерпретируемая модель с F1 = 0.77, не переобучается.
- Лучший feature set — не полный, а отобранный: robust_D для деревьев, shortlist для линейной модели.
- Модель, идеально выучившая train-данные (включая шумы), на новых данных ошибается — поэтому выбор строится по стабильности на валидации, а не по train-метрике.

---

## 7. Проверка понимания

1. **Нельзя выбирать гиперпараметры по test**, потому что test должен имитировать невидимые данные — подстройка под него даёт завышенную оценку обобщения.
2. **Train-метрика выше**, потому что модель видела эти примеры и могла выучить шумы; validation — новые данные, поэтому качество обычно ниже.
3. **Pipeline внутри CV** гарантирует, что масштабирование и one-hot кодирование обучаются только на train-фолдах. Однако он не защищает от утечки, если отбор признаков с участием validation был сделан до CV.
4. **Более сложная модель может не дать выигрыша**, если базовая уже оптимальна (например, логистическая регрессия на линейно разделимых данных) или если данных недостаточно.
5. **Didactic shortcut:** в этой работе один и тот же validation использовался для выбора feature set, построения validation curves и финального сравнения. В продакшне необходим отдельный selection split или nested CV.

---

## 8. Возможные улучшения

- Добавить третью модель (LinearSVC) и сравнить с текущими.
- Использовать RandomizedSearchCV для экономии времени при большом пространстве параметров.
- Провести nested CV для более честной оценки обобщения.
- Изучить влияние разных `random_state` на стабильность результатов.
