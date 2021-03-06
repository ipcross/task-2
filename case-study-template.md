# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо было обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

Она успешно работала на файлах размером пару мегабайт, но для большого файла она работала слишком долго, и не было понятно, закончит ли она вообще работу за какое-то разумное время.

Я решил исправить эту проблему, оптимизировав эту программу.

## Формирование метрики
Для того, чтобы понимать, дают ли мои изменения положительный эффект на быстродействие программы буду использовать такую метрику:
- Количество итераций в секунду (ips) выполнения программы на данных размером 0,25Мb

Время выполнения исходного кода на файлах разной величины:
```
Calculating -------------------------------------
    Process 0.0625Mb      7.227  (±13.8%) i/s -     35.000  in   5.016108s
     Process 0.125Mb      1.793  (± 0.0%) i/s -      9.000  in   5.179667s
      Process 0.25Mb      0.504  (± 0.0%) i/s -      3.000  in   5.975735s
       Process 0.5Mb      0.146  (± 0.0%) i/s -      1.000  in   6.864431s
         Process 1Mb      0.039  (± 0.0%) i/s -      1.000  in  25.710734s
         Process 2Mb      0.009  (± 0.0%) i/s -      1.000  in 109.083634s

Comparison:
    Process 0.0625Mb:        7.2 i/s
     Process 0.125Mb:        1.8 i/s - 4.03x  slower
      Process 0.25Mb:        0.5 i/s - 14.34x  slower
       Process 0.5Mb:        0.1 i/s - 49.61x  slower
         Process 1Mb:        0.0 i/s - 185.80x  slower
         Process 2Mb:        0.0 i/s - 788.29x  slower
```
Тенденция: при увеличении обьема исходных данных в два раза, время заметляется в 4 раза.

Функция растет очерь быстро, надо разбираться с алгоритмом.

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста позволяет не допустить изменения логики программы при оптимизации.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`, который позволил мне получать обратную связь по эффективности сделанных изменений за время ~5 секунд.

Вот как я построил `feedback_loop`:
- тестовый файл 0,25Мb, исходное время выполнения 5 секунд
- поиск самого узкого места
- улучшение кода
- замеры метрик
- запуск тестов
- анализ результатов

Исходная программа имеет метрику **~0.504ips**

## Вникаем в детали системы, чтобы найти 20% точек роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался библиотеками benchmark, ruby-prof, Valgrind, stackprof.

Вот какие проблемы удалось найти и решить

### Valgrind massif
Профиль использования памяти на файле 0,25Мb
`valgrind --tool=massif ruby task-2.rb`
![image massif](https://a.radikal.ru/a22/1903/7f/47d630ea9c4d.jpg)

### Ваша находка №1
Профилирую программу с помощью **stackprof**
```
 8856   (94.8%)                   |    99  |   users.each do |user|
                                  |   100  |     attributes = user
 17572  (188.1%) /  8786  (94.1%)  |   101  |     user_sessions = sessions.select { |session| session['user_id'] == user['id'] }
   35    (0.4%)                   |   102  |     user_object = User.new(attributes: attributes, sessions: user_sessions)
                                  |   103  |     users_objects = users_objects + [user_object]
   35    (0.4%) /    35   (0.4%)  |   104  |   end
```
Видим, что больше всего ресурсов тратится из-за неоптимальности алгоритма. Решение, переписать его с пользованием хеша.

#### Эффект изменения
Метрика выросла с `~0.504ips` до `~8.4ips`.

### Ваша находка №2
Смотрим отчет **memory_profiler**
Больше всего памяти выделяется при конкатенации массивов.
Решение, замена конкатенации на модификацию массивов `users` и `sessions` 'in place' с помощью `Array#push`

#### Эффект изменения
Метрика сильно не изменилась.

### Ваша находка №3
Удалось добиться небольшого ускорения, собрав всю статистику пользователя за одну итерацию (вместо нескольких для каждого вида статистики).

### Ваша находка №4
Оптимизация регулярок (замена `=~` на `match?`, вынос регулярного выражения вне цикла)

### Ваша находка №5
Смотрим отчет **ruby_prof** в режиме **WALL_TIME** отчет CallStack
```
42.25% (83.47%) <Class::Date>#parse [5595 calls, 5595 total]
      12.09% (28.61%) String#gsub! [5595 calls, 5595 total]
      4.79% (11.34%) Regexp#match [11190 calls, 11190 total]
      3.05% (7.23%) MatchData#begin [5595 calls, 5595 total]
      1.46% (3.46%) String#[]= [5595 calls, 5595 total]
3.10% (6.13%) Date#iso8601 [5595 calls, 5595 total]
```
Нет нужды использовать Date объект, дата уже в правильном формате. Удаляем лишнее.

#### Эффект изменения
Метрика выросла до `~29.5ips`.

### Ваша находка №6
Смотрим отчет **ruby_prof** в режиме **WALL_TIME** отчет CallStack
45.18% (45.20%) <Class::IO>#foreach [1 calls, 1 total]
Чтение файла построчно позволяет сэкономить немного памяти и ускорить работу.

### Ваша находка №7
Смотрим отчет **ruby_prof** в режиме **WALL_TIME** отчет CallStack
```
8.01% (18.02%) JSON::Ext::Generator::GeneratorMethods::Hash#to_json [1 calls, 1 total]
      6.62% (36.77%) String#encode [16909 calls, 16909 total]
      1.83% (10.15%) String#to_s [8224 calls, 8224 total]
```
Заменяем стандартную библиотеку json на гем oj.

#### Эффект изменения
Метрика выросла до `~33ips`.


## Результаты
В результате проделанной оптимизации наконец удалось обработать файл с данными.
Удалось улучшить метрику системы:

Для тестового файла до **33.5**
```
Calculating -------------------------------------
Process 0.25 MB of data
                         33.598  (± 6.0%) i/s -    168.000  in   5.017205s
```
Для основного файла:
```
Calculating -------------------------------------
Process 129 MB of data
                         0.040  (± 0.0%) i/s -      1.000  in  25.225046s
```
Асимптотика:
```
Calculating -------------------------------------
    Process 0.0625Mb    149.586  (±10.0%) i/s -    737.000  in   5.004075s
     Process 0.125Mb     75.342  (±15.9%) i/s -    363.000  in   5.007833s
      Process 0.25Mb     32.027  (± 9.4%) i/s -    159.000  in   5.008409s
       Process 0.5Mb     15.482  (± 6.5%) i/s -     78.000  in   5.046184s
         Process 1Mb      8.162  (±12.3%) i/s -     41.000  in   5.051811s
         Process 2Mb      3.812  (± 0.0%) i/s -     19.000  in   5.015254s

Comparison:
    Process 0.0625Mb:      149.6 i/s
     Process 0.125Mb:       75.3 i/s - 1.99x  slower
      Process 0.25Mb:       32.0 i/s - 4.67x  slower
       Process 0.5Mb:       15.5 i/s - 9.66x  slower
         Process 1Mb:        8.2 i/s - 18.33x  slower
         Process 2Mb:        3.8 i/s - 39.24x  slower
```

### Valgrind massif
Профиль использования памяти на исходном файле
![image massif](https://c.radikal.ru/c28/1903/67/0fd8169c033a.jpg)

## Защита от регресса производительности
Для защиты от потери достигнутого прогресса при дальнейших изменениях программы добавлен regress тест.
