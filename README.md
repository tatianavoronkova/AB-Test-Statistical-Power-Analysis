# Анализ статистической мощности. Оценка вероятности обнаружения эффекта методом Монте-Карло.
## Входные данные:
В приложении есть сервис ленты новостей. Команда ML-отдела планируют развернуть новый алгоритм, рекомендующий пользователям интересные посты. 
Основные фичи:
- Алгоритм добавляет пользователям **1-2 просмотра**
- **Вероятность** того, что он сработает, **составляет 90%**
- Если у пользователя **меньше 50 просмотров**, то алгоритм **не сработает**
  
Предполагается, что увеличение числа просмотров приведёт к увеличению лайков на пользователя.

Требуется проверить, **возможно ли обнаружить различия в среднем количестве лайков на пользователя** во время А/В тестирования нового алгоритма? 
Другими словами, нужно рассчитать статистическую мощность до проведения эксперимента, чтобы определить вероятность обнаружения истинного эффекта (если он существует) с учетом заданного уровня значимости и размера выборки. Чтобы ответить на этот вопрос, проведём **симуляцию Монте-Карло**.

**Период эксперимента:** 1 неделя  
**Уровень значимости alpha** = 0.05  
**Размер выборок:** количество пользователей в период А/А теста с 2024-04-26 по 2024-05-02 в соотношении 50/50  

См. задачу ["Анализ результатов А/А теста. Проверка корректности работы системы сплитования."](https://github.com/tatianavoronkova/AA-Test-Splitting-System-Analysis)

## Задача: рассчитать статистическую мощность 
1. Зафиксировать количество тестов, которые будем проводить (N = 20,000)
2. Посчитать размер выборок из расчета (кол-во пользователей в период А/А теста) * 0.5
3. Посчитать вероятности каждого кол-ва просмотра за период А/А теста
4. Сгенерировать выборки просмотров группы А и В с учетом рассчетной вероятности из п.3
5. Наложить эффект алгоритма на просмотры группы В:  
     *group_B_views + ((1 + np.binomial(n=1, p=0.5, size=размер_выборки)) * np.binomial(n=1, p=0.9, size=размер_выборки) * (group_B_views >= 50))*
6. Посчитать вероятности CTR (click-through-rate) для групп А и В, чтобы использовать это как оценку вероятности лайка
7. Получить выборку лайков из биномиального распределения (просмотры, CTR) для групп А и В
8. Сравнить выборки лайков N раз с использованием t-теста с поправкой Уэлча на неравные дисперсии (equal_var=False) и уровнем значимости 0.05
9. Посчитать мощность теста

## Результат:
<image width="500" height="100" src="/images/Stat_power_ver01.png" alt="Statistical power ver 01">

Мощность теста 26% - слишком мала вероятность обнаружить эффект алгоритма.
Для увеличения мощности рекомендуется:
- улучшить качество тестируемого алгоритма для усиления ожидаемого эффекта
- увеличить размер выборки или изменить качество выборки
- увеличить alpha (риск ошибки 1 рода)

# Реализация рекомендаций

## Корректировка №1
ML улучшили качество алгоритма, теперь он срабатывает на пользователях с числом просмотров от 30 и выше, т.е:
 - Алгоритм добавляет пользователям 1-2 просмотра
 - Вероятность того, что он сработает, составляет 90%
 - Если у пользователя **меньше 30 просмотров**, то алгоритм не сработает

### Задача: подкорректировать симуляцию и посчитать новое значение мощности
1. Изменить эффект алгоритма на просмотры группы В:  
     *group_B_views + ((1 + np.binomial(n=1, p=0.5, size=размер_выборки)) * np.binomial(n=1, p=0.9, size=размер_выборки) * (group_B_views >= 30))*
2. Посчитать мощность

### Результат
<image width="500" height="100" src="/images/Stat_power_ver02.png" alt="Statistical power ver 02">
Мощность теста 42% - лучше, но по-прежнему мало, чтобы верно обнаружить эффект от алгоритма.

## Корректировка №2
Длительность эксперимента увеличили до 2 недель. Предполагается, что за две недели в приложение зайдет столько же пользователей, сколько пришло суммарно за период АА-теста и АБ-теста с 2024-04-26 по 2024-05-09.

### Задача: увеличить выборку и посчитать новое значение мощности
1. Подкорректировать размер выборки из расчета (кол-во пользователей в период А/А теста и А/В теста) * 0.5
2. Сами распределения, на основе которых мы симулируются данные, не изменяются
3. Посчитать мощность

### Результат
<image width="500" height="100" src="/images/Stat_power_ver03.png" alt="Statistical power ver 03">
Мощность теста 56% - лучше, но по-прежнему мало, чтобы верно обнаружить эффект от алгоритма.

## Корректировка №3
В выборке есть пользователи, которых не затронет действие ногово алгоритма (меньше 30 просмотров). Изменим качество выборки и оставим только активных пользователей, на которых алгоритм повлияет. Ожидается, что чувствительность теста вырастет, несмотря на то, что размер выборки уменьшится.

### Задача: отфильтровать выборку и посчитать новое значение мощности
1. Применить фильтр (симулируем данные просмотров, CTR и лайков):
  _mask_A = group_A_views >= 30_  
  _mask_B = group_B_views >= 30_  
  scipy.stats.ttest_ind(group_A_likes[mask_A], group_B_likes[mask_B], equal_var=False)
2. Посчитать мощность

### Результат
<image width="500" height="100" src="/images/Stat_power_ver04.png" alt="Statistical power ver 04">
  
**Мощность теста 64%** - хороший результат, но так и не дотягивает до целевого значения 80%
