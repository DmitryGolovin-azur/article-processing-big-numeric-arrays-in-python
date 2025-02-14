# Как быстро и удобно ворочать большими числовыми массивами в Python (часть 1)

## Введение
В статье речь пойдет о базовых технических аспектах обработки больших числовых массивов в Python. Статья ориентирована на начинающих data scientists, которые используют Python. Я покажу основные подходы и инструменты, но не буду сильно останавливаться на деталях (они есть в документации), иначе статья получится слишком большой. =)

На каких данных начать тренироваться новичку? Я считаю хорошим источником данные рынка акций. Источников этих данных предостаточно (где и как их найти я расскажу в конце статьи) и это просто рай для начинающих ученых в области данных. А самое главное, они “живые”, можно разработать стратегию управления портфелем и проверить ее, подождав несколько дней. Мы будем использовать финансовые данные рынка американских акций как источник.

Будем разрабатывать простейшую инвестиционную стратегию на основе скользящих средних (ema), но на довольно больших объемах данных.

Мы сосредоточимся на технических аспектах реализации одной из инвестиционных стратегий.

## Итак, задача
Возьмем гипотетическую задачу по реализации инвестиционной стратегии.

Дано: финансовые данные (open, high, low, close, volume, split, divs) ~2000 акций за последние ~20 лет.

Нужно рассчитать распределение весов акций в портфеле для каждого дня, если разрешено покупать акции только с положительным ema(20) от close, а соотношение весов должно соответствовать соотношению ema(30) от волатильности (close*volume).

EMA - экспоненциальное скользящее среднее (https://en.wikipedia.org/wiki/Moving_average#Exponential_moving_average).

Ограничения: расчеты должны занимать менее 10 секунд и потреблять менее 1.5 GB памяти.

Сильно не заморачивайтесь на самой задаче (на том, что нужно посчитать). Она довольно тривиальная и нужна только для примера. Обращайте внимание на технические аспекты.

## Грузим данные
### В лоб
Допустим, вы каким-то образом получили эти данные в виде ~2000 csv файлов по каждой акции (где именно и как получить данные я расскажу в конце статьи), в следующем формате:

```
date,open,high,low,close,vol,divs,split
1998-01-01,2,3,1,2,23232,0.1,1
1998-01-02,2,3,1,3,32423,0,2
...
```
 
Первый шаг - это загрузить данные в память для дальнейшей обработки. Тот, кто только что изучил Python, сразу ломанется писать что-то наподобие:

[Example #1. Code](../src/p1_e1_load.py)
```python
import os
import csv
from datetime import datetime

DATA_DIR='../data'
# this directory contains only 1 sample,
# make multiple copies of this sample for reasonable results

data = dict()
for root, dirs, files in os.walk(DATA_DIR):
    for filename in files:
        asset = filename.split('.')[0]
        file_path = os.path.join(DATA_DIR, filename)
        with open(file_path) as csvfile:
            csvreader = csv.reader(csvfile, delimiter=',')
            first = True
            series = []
            headers = None
            for row in csvreader:
                if first:
                    first = False
                    headers = row
                    continue
                row = [float(row[i]) if i > 0 else datetime.strptime(row[i], "%Y-%m-%d").date()
                       for i in range(len(row))]
                series.append(row)
            data[asset] = series

print("loaded")

# меряем память после загрузки массива данных
from memory_profiler import memory_usage
print(memory_usage())
``` 

Попробуем это запустить (заодно замеряем потребление ОЗУ и время):

[Example #1. Run](../src/p1_e1_run.sh)
```bash
#!/usr/bin/env bash

echo "Part #1 example #1: load data from csv"
/usr/bin/time -f "%E %MKb" python3 p1_e1_load.py > ../report/p1_e1.txt 2>&1
```

[Example #1. Report](../report/p1_e1.txt)
```
loaded
[3462.78515625]
2:01.76 3545892Kb
```

Только загрузка сожрала ~3.5GB памяти и заняла около 2 минут. 
Это у меня-то, с Intel SSD 760p и процессором Intel i5 8300H!!!1 
А ведь суммарный размер этих файлов на жестком диске всего около 0.6GB!

### Что так долго и куда столько памяти?
Ответ на первую часть - парсинг чисел и дат на Python (да и вообще обработка больших массивов данных) далеко не самая быстрая операция. Парсить такие объемы очень долго, нужно что-то пошустрее.

Ответ на вторую часть: каждый объект python сам по себе жрет довольно много памяти, а ведь каждое число или дата - это объект. Даже если все затолкать в один линейный список, по памяти все равно не уложиться.

Ах, вот если бы можно было использовать числовые массивы напрямую размещенные в памяти как в С!… C?!.. Oh, shi… У нас есть возможность использовать такие массивы, для этого есть библиотека numpy и ее ndarray (https://docs.scipy.org/doc/numpy/reference/generated/numpy.ndarray.html).

numpy.ndarray позволяет размещать числовые (и не только числовые) массивы с элементами одного типа напрямую в памяти. Эти массивы, размещенные напрямую в памяти тем и хороши, что для их работы требуется мало накладных расходов (числа укладываются последовательно друг за другом в памяти без дополнительных расходов). По моим прикидкам это должно быть примерно 0.5GB (если использовать double)

### pandas.DataFrame
Но не спеши хватать голый ndarray. Как же быть с датами? Для удобной работы с датами нужен индекс по датам. Если погуглить какое-нибудь решение для ndarray и time series, то можно очень быстро наткнуться на pandas.DataFrame. pandas.DataFrame базируется на numpy.ndarrays, штука довольно удобная и говорят, хорошо работает с датами в качестве основного индекса. До кучи, у этой библиотеки есть какая-то встроенная поддержка загрузки CSV и все рекомендуют. Ништяк, берем эту либу и переписываем код загрузки данных:

[Example #2. Code](../src/p1_e2_load_pd.py)
```python
import os
import pandas as pd

DATA_DIR='../data'

data = dict()
for root, dirs, files in os.walk(DATA_DIR):
    for filename in files:
        asset = filename.split('.')[0]
        file_path = os.path.join(DATA_DIR, filename)
        series = pd.read_csv(file_path, sep=",", index_col="date")
        data[asset] = series

print("loaded")

# measure memory after loading
from memory_profiler import memory_usage
print(memory_usage())
``` 

Попробуем это запустить, заодно замеряем память и время:

[Example #2. Run](../src/p1_e2_run.sh)
```bash
#!/usr/bin/env bash

echo "Part #1 example #2: load data with pandas from csv"
/usr/bin/time -f "%E %MKb" python3 p1_e2_load_pd.py > ../report/p1_e2.txt 2>&1
```

[Example #2. Report](../report/p1_e2.txt)
```
loaded
[1383.421875]
0:15.42 1416848Kb
```

Что-то около 16 секунд и 1.3GB памяти (???). 
Памяти и времени сожрало хоть и меньше, но все равно довольно много. 
А ведь даже вычислений еще и не делали, так не пойдет. 
Давайте разберемся.

Эти накладные расходы связаны с тем, что индексы в pandas дублируются и это получается довольно накладно. Также, при разработке финансовых стратегий, часто приходится выравнивать данные, чтобы индексы были одинаковыми. Возникает желание объединить данные в один многомерный массив, для чего в pandas был Panel, но он устарел...

### xarray.DataArray
На замену pandas.Panel была разработана другая библиотека, которы называется она xarray.DataArray (http://xarray.pydata.org/en/stable/generated/xarray.DataArray.html). Почему-то, незаслуженно, люди используют ее гораздо реже. Если сказать честно, эта надстройка в некоторы случаях даже удобнее чем pd.DataFrame для вычислений (что я покажу позже).

Будем использовать xarray.DataArray, чтобы загрузить данные в большой 3-мерный массив.
Почитав еще немного доки, узнаем что для загрузки и сохранения данных в xarray можно использовать 
бинарный формат NetCDF (scipy). Ништяк, пишем конвертер данных из кучи файлов CSV в один многомерный NetCDF 
(а я и не говорил, что данные обязательно в CSV оставлять 8-) ).

[Example #3. Code](../src/p1_e3_convert_to_nc.py)
```python
import os
import pandas as pd
import xarray as xr

DATA_DIR='../data'
NETCDF_FILE = '../data.nc'

data = []
for root, dirs, files in os.walk(DATA_DIR):
    for filename in files:
        asset = filename.split('.')[0]
        file_path = os.path.join(DATA_DIR, filename)
        series = pd.read_csv(file_path, sep=",", index_col="date")
        series = series.to_xarray().to_array('field')
        series.name = asset
        data.append(series)
        series = None

data = xr.concat(data, pd.Index([d.name for d in data], name='asset'))
data.to_netcdf(NETCDF_FILE, compute=True)
``` 

[Example #3. Run](../src/p1_e3_run.sh)
```bash
#!/usr/bin/env bash

echo "Part #1 example #3: convert data from csv to netcdf"
/usr/bin/time -f "%E %MKb" python3 p1_e3_convert_to_nc.py > ../report/p1_e3.txt 2>&1
```

Запускаем его, смотрим что получилось. На выходе получился один файл data.nc объемом примерно те же 0.6GB. Ок, грузим его в xarray.

[Example #4. Code](../src/p1_e4_load_xr_netcdf.py)
```python
import xarray as xr

data = xr.open_dataarray('../data.nc', decode_times=True).compute()

print("loaded")

# measure memory after loading
from memory_profiler import memory_usage
print(memory_usage())
``` 

[Example #2. Run](../src/p1_e4_run.sh)
```bash
#!/usr/bin/env bash

echo "Part #1 example #4: load data with xarray from netcdf"
/usr/bin/time -f "%E %MKb" python3 p1_e4_load_xr_netcdf.py > ../report/p1_e4.txt 2>&1
```

[Example #2. Report](../report/p1_e4.txt)
```
loaded
[648.6171875]
0:00.93 1242180Kb
```

Вот так уже норм. Загрузка в пике схавала 1.2GB, но после загрузки в памяти массив данных занял 0.6GB (почти в 2 раза меньше). Это чуть больше, чем я рассчитывал, но приемлемо. Времени это заняло около секунды. Идем дальше с xarray.

Небольшой комментарий. У нас данные все одного типа - double (кроме дат и названий ассетов, но они выступают индексами), потому их и удобно уложить в xarray.DataArray. Но если данные были бы разных типов (разные типы для каждого столбца field), то, возможно, удобнее было бы использовать xarray.DataSet (это логическое продолжение DataFrame для более чем 2 измерений) или несколько DataArray. Например, volume можно уложить в целочисленный массив (но для дальнейшего анализа и вычислений это не имеет особого смысла). pandas.DataFrame, хорошо подходит для данных, где столбцы имеют разный тип и вам нужно только 2 измерения. Однако у нас их три (название тикера, дата, столбец данных). Для более обдуманного применения надо знать как устроены все эти структуры (numpy.ndarray, pandas.Series, pandas.DataFrame, xarray.DataArray, xarray.DataSet и т.д.).

И еще один: в принципе, вместо NetCDF можно использовать Pickle. Но у него некоторые проблемы с переносимостью. 
Т.е. если вы сохранили xarray.DataArray версии .15 и загружаете его библиотекой версии .16, это может стать проблемой.
Короче, для долгосрочного хранения данных он не подходит, но для краткосрочного - ок.

## Промежуточный итог
На этом я вынужден прерваться, чтобы эта часть сильно не распухла. Подведем промежуточный итог. Мы уменьшили время загрузки данных с 100 секунд до 1 и потребляемую память ОЗУ с 3.4 GB до 1.2 в пике или 0.6 GB после загрузки данных.

Почему так важно правильно грузить данные? Да потому, что, разрабатывая свои алгоритмы, вы будете сотни (или тысячи) раз запускать свои программы, а если вы будете тратить при этом каждый раз 100 секунд, это, мягко говоря, накладно. Опять же, если вы меньше тратите памяти впустую, то можете больше полезных данных натолкать для расчетов (в 3-6 раз). Следующий момент не так очевиден, но он означает, что используя ndarray (pandas и xarray надстройки над ним), мы также получаете возможность проводить вычисления гораздо быстрее чем на “чистом python”. Я покажу это во второй части статьи. Это ~~мелкие убогие фокусы~~ “пар и трубы”, как говорят сантехники, на этом работает data science в python.

## Выводы
1. Используйте специализированные бинарные форматы, чтобы ускорить загрузку данных (их не надо парсить, а значит загрузка будет быстрее).

2. Собирайте кучу маленьких файлов в большие файлы. Один файл размером в половину гигабайта будет грузиться сильно быстрее тысяч мелких файлов.

3. Используйте специальные библиотеки для загрузки данных и хранения их в ОЗУ. Это опять же экономит время и память.

## PS
### Э, как же стриминг?
Особо умные могут сказать, что грузить все данные в память не надо, а надо читать из файлов по мере потребности.
На что я резонно могу возразить:

Это намного медленнее. Считать все сразу конечно же быстрее.
Это неудобнее. Алгоритм придется писать, подстраиваясь под ограничение стриминга.

### А вдруг не влезет?
А если кто-то приведет довод, что все данные могут не уместиться в памяти, то я могу заметить, что чаще используется другой подход, когда данные бьются на относительно большие куски которые можно обработать отдельно, а потом склеивают результат обработки. В этом случае обработка отдельного куска не сильно отличаться от того, что мы имеем здесь. Так же: загрузил кусок, посчитал, сохранил.

Но, возможно, вам и не придется ничего бить на куски. Если вы сэкономите память, то, вполне возможно, сможете затолкать в нее весь массив данных сразу.

## Где продолжение?!
Следующая часть статьи будет опубликована на следующей неделе, там я покажу, как быстро обрабатывать такие массивы данных. Чтоб подогреть интерес, я скажу, что разница между наивным и эффективным подходом там еще на порядок больше.

Ну все, теперь ждите и страдайте от предвкушения.

{ % include ../comments.html }
