
Достаточно частый вопрос у пользователей компонента, как вывести рейтинг при использовании сниппета msProducts (или pdoResources).

У нас есть 2 варианта:

**Вариант 1.** Отдельный вызов сниппета ecThreadRating в чанке вывода одного товара (напоминаю, по-умолчанию используется tpl.msProducts.row).

Здесь главное правильно сформировать параметр thread, на синтаксисе родного парсера MODX это будет

```php
&thread=`resource-[[+id]]`
```

, а на синтаксисе Fenom

```php
'thread' => 'resource' ~ $id,
```

Этот вариант прост, но при выводе большого количества товаров на одной странице мы получим большое количество запросов в базу данных. Можно столкнуться с падением скорости работы сайта.

**Вариант 2.** Использовать JOIN, чтобы одним запросом выбирать нужные данные.

У нас есть пара вариантов того, с использованием какого поля присоединять таблицы:

- поле resource_id объекта ecThread, указывает соответственно на товар (или страницу);
- поле name объекта ecThread так же содержит в себе id товара (например resource-42).

Если у вас одному товару соответствует одна цепочка (например отзывы), то можно сделать JOIN с использованием первого варианта:

```php
&loadModels=`easycomm`
&leftJoin=`{
    "ecThread": {
        "class": "ecThread",
        "on": "msProduct.id = ecThread.resource_id"
    }
}`
&select=`{
    "msProduct": "*",
    "ecThread": "ecThread.rating_simple AS rating"
}`
```

Обращаю внимание, в select мы указали, что выбираем только поле rating_simple как rating, нужно больше - или пишите *, или указывайте нужные поля.

Код выше плохо сработает (а точнее 100% не сработает), если у вас одному товару соответствует несколько (две и более) цепочек, к примеру у вас есть Отзывы и Вопрос-Ответ. Цепочки 2, но мы явно не указываем, какую использовать в нашем JOIN, получим некорректные данные.

Решение - делать выборку с использованием поля ecThread.name. В примере ниже мы вообще из одной цепочки узнаем рейтинг (Rating), а из другой цепочки мы узнаем кол-во вопросов по этому товару (QA). В этом примере имена цепочек формируются как resource-rating-XX и resource-qa-XX. 

```php
&loadModels=`easycomm`
&leftJoin=`{
        "ecThreadRating": {
        "class": "ecThread",
        "alias": "ecThreadRating",
        "on": " CONCAT('resource-rating-', modResource.id) = ecThreadRating.name"
    },
        "ecThreadQA": {
        "class": "ecThread",
        "alias": "ecThreadQA",
        "on": " CONCAT('resource-qa-', modResource.id) = ecThreadReviews.name"
    }
}`
&select=`{
    "modResource": "*",
    "ecThreadRating": "ecThreadRating.rating_simple AS rating",
    "ecThreadQA": "ecThreadQA.count AS qa_count"
}`
```

Как видно, мы используем оператор CONCAT, чтобы сформировать с использованием id товара имя цепочки и выборку ограничиваем им.
