# Глава 7. Улучшения и добавление аутентификации в API в Flask

В этой главе мы улучшим RESTful API, который мы начали в предыдущей главе, и добавим к нему безопасность, связанную с аутентификацией. Мы будем:

* Улучшать уникальные ограничения в моделях
* Обновлять поля для ресурса методом PATCH
* Создавать код универсального класса разбиения на страницы
* Добавлять функции разбивки на страницы в API
* Изучать шаги по добавлению проверки подлинности и разрешений.
* Добавлять модель пользователя
* Создавать схему для проверки, сериализации и десериализации пользователей.
* Добавлять аутентификацию к ресурсам
* Создавать классы ресурсов для обработки пользователей
* Запускать миграции для создания пользовательской таблицы
* Составлять запросы с необходимой аутентификацией

## Улучшение уникальных ограничений в моделях

При создании модели **Category** мы указали значение `True` для аргумента **unique** при создании экземпляра **db.Column** с именем **name**. В результате миграция создала необходимое уникальное ограничение, чтобы убедиться, что поле **name** имеет уникальные значения в таблице **category**. Таким образом, база данных не позволит нам вставлять повторяющиеся значения для `category.name`. Однако сообщение об ошибке, сгенерированное, когда мы пытаемся это сделать, неясно.

Выполните следующую команду, чтобы создать категорию с дубликатом имени. Уже существует категория с названием `'Information'`:

```bash
http POST :5000/api/categories/ name='Information'
```

Ниже приведена эквивалентная команда **curl**:

```bash
curl -iX POST -H "Content-Type: application/json" -d '{"name":"Information"}'
:5000/api/categories/
```

Предыдущая команда создаст и отправит HTTP-запрос **POST** с указанной парой ключ-значение **JSON**. Уникальное ограничение в поле `category.name` не позволит таблице базы данных сохранить новую категорию. Таким образом, запрос вернет код состояния HTTP **400 Bad Request** с сообщением об ошибке целостности. В следующих строках показан пример ответа:

```http
HTTP/1.0 400 BAD REQUEST
Content-Length: 282
Content-Type: application/json
Date: Mon, 15 Aug 2016 03:53:27 GMT
Server: Werkzeug/0.11.10 Python/3.5.1
{
    "error": "(psycopg2.IntegrityError) duplicate key value violates unique
    constraint "category_name_key"\nDETAIL: Key (name)=(Information)
    already exists.\n [SQL: 'INSERT INTO category (name) VALUES (%(name)s)
    RETURNING category.id'] [parameters: {'name': 'Information'}]"
}
```

Очевидно, что сообщение об ошибке носит исключительно технический характер и содержит слишком много подробностей о базе данных и запросе, который завершился ошибкой. Мы можем проанализировать сообщение об ошибке, чтобы автоматически сгенерировать более удобное для пользователя сообщение об ошибке. Однако вместо этого мы хотим избежать попытки вставить строку, которая, как мы знаем, не удастся. Мы добавим код, чтобы убедиться, что категория уникальна, прежде чем мы попытаемся ее сохранить. Конечно, все еще есть шанс получить показанную ранее ошибку, если кто-то вставит категорию с тем же именем между запуском нашего кода, указывая, что имя категории уникально, и сохранит изменения в базе данных. Однако этот шанс ниже, и мы можем уменьшить изменения отображаемого ранее сообщения об ошибке.

{% hint style="info" %}
**СОВЕТ**

В готовом к работе REST API мы никогда не должны возвращать сообщения об ошибках, возвращаемые **SQLAlchemy**, или любые другие данные, связанные с базой данных, поскольку они могут включать конфиденциальные данные, которые мы не хотим, чтобы пользователи нашего API могли извлекать. В этом случае мы возвращаем все ошибки в целях отладки и для улучшения нашего API.
{% endhint %}

Теперь мы добавим новый метод класса в класс **Category**, чтобы определить, является ли имя уникальным или нет. Откройте файл `api/models.py` и добавьте следующие строки в объявление класса **Category**. Файл кода для примера включен в папку `restful_python_chapter_07_01`:

```python
@classmethod
def is_unique(cls, id, name):
    existing_category = cls.query.filter_by(name=name).first()
    if existing_category is None:
        return True
    else:
        if existing_category.id == id:
            return True
        else:
            return False
```

Новый метод класса `Category.is_unique` получает идентификатор **id** и имя категории **name**, для которой мы хотим убедиться, что она имеет уникальное имя. Если категория новая, еще не сохраненная, мы получим `0` для значения **id**. В противном случае мы получим идентификатор категории в аргументе.

Метод вызывает метод `query.filter_by` для текущего класса, чтобы получить категорию, имя которой совпадает с именем другой категории. В случае наличия категории, соответствующей критериям, метод вернет `True` только в том случае, если идентификатор совпадает с полученным в аргументе. Если ни одна категория не соответствует критериям, метод вернет значение `True`.

Мы будем использовать ранее созданный метод класса, чтобы проверить, является ли категория уникальной или нет, прежде чем создавать и сохранять ее в методе `CategoryListResource.post`. Откройте файл `api/views.py` и замените существующий метод **post**, объявленный в классе **CategoryListResource**, следующими строками. Строки, которые были добавлены или изменены, выделены. Файл кода для примера включен в папку `restful_python_chapter_07_01`:

```python
def post(self):
    request_dict = request.get_json()
    if not request_dict:
        resp = {'message': 'No input data provided'}
        return resp, status.HTTP_400_BAD_REQUEST
    errors = category_schema.validate(request_dict)
    if errors:
        return errors, status.HTTP_400_BAD_REQUEST
    category_name = request_dict['name']
    if not Category.is_unique(id=0, name=category_name):
        response = {'error': 'A category with the same name already exists'}
        return response, status.HTTP_400_BAD_REQUEST
    try:
        category = Category(category_name)
        category.add(category)
        query = Category.query.get(category.id)
        result = category_schema.dump(query).data
        return result, status.HTTP_201_CREATED
    except SQLAlchemyError as e:
        db.session.rollback()
        resp = {"error": str(e)}
        return resp, status.HTTP_400_BAD_REQUEST
```

Теперь мы выполним ту же проверку в методе `CategoryResource.patch`. Откройте файл `api/views.py` и замените существующий метод **patch**, объявленный в классе **CategoryResource**, следующими строками. Строки, которые были добавлены или изменены, выделены. Файл кода для примера включен в папку `restful_python_chapter_07_01`:

```python
def patch(self, id):
    category = Category.query.get_or_404(id)
    category_dict = request.get_json()
    if not category_dict:
        resp = {'message': 'No input data provided'}
        return resp, status.HTTP_400_BAD_REQUEST
    errors = category_schema.validate(category_dict)
    if errors:
        return errors, status.HTTP_400_BAD_REQUEST
    try:
        if 'name' in category_dict:
            category_name = category_dict['name']
            if Category.is_unique(id=id, name=category_name):
                category.name = category_name
            else:
                response = {'error': 'A category with the same name already exists'}
                return response, status.HTTP_400_BAD_REQUEST
        category.update()
        return self.get(id)
    except SQLAlchemyError as e:
        db.session.rollback()
        resp = {"error": str(e)}
        return resp, status.HTTP_400_BAD_REQUEST
```

Выполните следующую команду, чтобы снова создать категорию с дубликатом имени:

```bash
http POST :5000/api/categories/ name='Information'
```

Ниже приведена эквивалентная команда **curl**:

```bash
curl -iX POST -H "Content-Type: application/json" -d '{"name":"Information"}'
:5000/api/categories/
```

Предыдущая команда создаст и отправит HTTP-запрос **POST** с указанной парой ключ-значение **JSON**. Внесенные нами изменения будут генерировать ответ с удобным для пользователя сообщением об ошибке, и мы не будем пытаться сохранить изменения. Запрос вернет код состояния HTTP **400 Bad Request** с сообщением об ошибке в теле **JSON**. Следующие строки показывают образец ответа:

```http
HTTP/1.0 400 BAD REQUEST
Content-Length: 64
Content-Type: application/json
Date: Mon, 15 Aug 2016 04:38:43 GMT
Server: Werkzeug/0.11.10 Python/3.5.1
{
    "error": "A category with the same name already exists"
}
```

Теперь мы добавим новый метод класса в класс **Message**, чтобы определить, является ли сообщение уникальным или нет. Откройте файл `api/models.py` и добавьте следующие строки в объявление класса **Message**. Файл кода для примера включен в папку `restful_python_chapter_07_01`:

```python
@classmethod
def is_unique(cls, id, message):
    existing_message = cls.query.filter_by(message=message).first()
    if existing_message is None:
        return True
    else:
        if existing_message.id == id:
            return True
        else:
            return False
```

Новый метод класса `Message.is_unique` получает идентификатор **id** и **message** для сообщения, которое мы хотим убедиться, что оно имеет уникальное значение для поля **message**. Если это новое сообщение, которое еще не было сохранено, мы получим `0` для значения идентификатора **id**. В противном случае мы получим идентификатор сообщения в аргументе.

Метод вызывает метод `query.filter_by` для текущего класса, чтобы получить сообщение, поле сообщения которого соответствует сообщению другого сообщения. В случае наличия сообщения, соответствующего критериям, метод вернет `True` только в том случае, если идентификатор совпадает с полученным в аргументе. Если ни одно сообщение не соответствует критериям, метод вернет `True`.

Мы будем использовать ранее созданный метод класса, чтобы проверить уникальность сообщения перед его созданием и сохранением в методе `MessageListResource.post`. Откройте файл `api/views.py` и замените существующий метод **post**, объявленный в классе **MessageListResource**, следующими строками. Строки, которые были добавлены или изменены, выделены. Файл кода для примера включен в папку `restful_python_chapter_07_01`:

```python
def post(self):
    request_dict = request.get_json()
    if not request_dict:
        response = {'message': 'No input data provided'}
        return response, status.HTTP_400_BAD_REQUEST
    errors = message_schema.validate(request_dict)
    if errors:
        return errors, status.HTTP_400_BAD_REQUEST
    message_message = request_dict['message']
    if not Message.is_unique(id=0, message=message_message):
        response = {'error': 'A message with the same message already exists'}
        return response, status.HTTP_400_BAD_REQUEST
    try:
        category_name = request_dict['category']['name']
        category = Category.query.filter_by(name=category_name).first()
        if category is None:
            # Create a new Category
            category = Category(name=category_name)
            db.session.add(category)
        # Now that we are sure we have a category
        # create a new Message
        message = Message(
            message=message_message,
            duration=request_dict['duration'],
            category=category)
        message.add(message)
        query = Message.query.get(message.id)
        result = message_schema.dump(query).data
        return result, status.HTTP_201_CREATED
    except SQLAlchemyError as e:
        db.session.rollback()
        resp = {"error": str(e)}
        return resp, status.HTTP_400_BAD_REQUEST
```

Теперь мы выполним ту же проверку в методе `MessageResource.patch`. Откройте файл `api/views.py` и замените существующий метод **patch**, объявленный в классе **MessageResource**, следующими строками. Строки, которые были добавлены или изменены, выделены. Файл кода для примера включен в папку `restful_python_chapter_07_01`:

```python
def patch(self, id):
    message = Message.query.get_or_404(id)
    message_dict = request.get_json(force=True)
    if 'message' in message_dict:
        message_message = message_dict['message']
        if Message.is_unique(id=id, message=message_message):
            message.message = message_message
        else:
            response = {'error': 'A message with the same message already exists'}
            return response, status.HTTP_400_BAD_REQUEST
    if 'duration' in message_dict:
        message.duration = message_dict['duration']
    if 'printed_times' in message_dict:
        message.printed_times = message_dict['printed_times']
    if 'printed_once' in message_dict:
        message.printed_once = message_dict['printed_once']
    dumped_message, dump_errors = message_schema.dump(message)
    if dump_errors:
        return dump_errors, status.HTTP_400_BAD_REQUEST
    validate_errors = message_schema.validate(dumped_message)
    if validate_errors:
        return validate_errors, status.HTTP_400_BAD_REQUEST
    try:
        message.update()
        return self.get(id)
    except SQLAlchemyError as e:
        db.session.rollback()
        resp = {"error": str(e)}
        return resp, status.HTTP_400_BAD_REQUEST
```

Выполните следующую команду, чтобы создать сообщение с повторяющимся значением поля сообщения:

```bash
http POST :5000/api/messages/ message='Checking temperature sensor'
duration=25 category="Information"
```

Ниже приведена эквивалентная команда **curl**:

```bash
curl -iX POST -H "Content-Type: application/json" -d '{"message":"Checking
temperature sensor", "duration":25, "category": "Information"}'
:5000/api/messages/
```

Предыдущая команда создаст и отправит HTTP-запрос **POST** с указанной парой ключ-значение **JSON**. Внесенные нами изменения будут генерировать ответ с удобным для пользователя сообщением об ошибке и не будут пытаться сохранить изменения в сообщении. Запрос вернет код состояния HTTP **400 Bad Request** с сообщением об ошибке в теле **JSON**. В следующих строках показан пример ответа:

```http
HTTP/1.0 400 BAD REQUEST
Content-Length: 66
Content-Type: application/json
Date: Mon, 15 Aug 2016 04:55:46 GMT
Server: Werkzeug/0.11.10 Python/3.5.1
{
"error": "A message with the same message already exists"
}
```

## Обновление полей ресурса методом PATCH

Как мы объясняли в [Главе 6](glava-6.-rabota-s-modelyami-sqlalchemy-i-api-s-giperssylkami-v-flask.md), "_Работа с моделями, SQLAlchemy и API-интерфейсами с гиперссылками во Flask_", наш API может обновлять одно поле для существующего ресурса, поэтому мы предоставляем реализацию метода **PATCH**. Например, мы можем использовать метод **PATCH**, чтобы обновить существующее сообщение и установить для его полей **print\_once** и **print\_times** значение `true` и `1`. Мы не хотим использовать метод **PUT**, потому что этот метод предназначен _**для замены всего сообщения**_. Метод **PATCH** предназначен для применения дельты к существующему сообщению, поэтому целесообразно просто изменить значение этих двух полей.

Теперь мы составим и отправим HTTP-запрос для обновления существующего сообщения, в частности, для обновления значения полей **print\_once** и **print\_times**. Поскольку мы просто хотим обновить два поля, мы будем использовать метод **PATCH** вместо **PUT**. Убедитесь, что вы заменили 1 идентификатором или первичным ключом существующего сообщения в вашей конфигурации:

```bash
http PATCH :5000/api/messages/1 printed_once=true printed_times=1
```

Ниже приведена эквивалентная команда **curl**:

```bash
curl -iX PATCH -H "Content-Type: application/json" -d '{"printed_once":"true",
"printed_times":1}' :5000/api/messages/1
```

Предыдущая команда создаст и отправит HTTP-запрос **PATCH** со следующими указанными парами ключ-значение **JSON**:

```json
{
    "printed_once": true,
    "printed_times": 1
}
```

Запрос имеет номер после `/api/messages/`, поэтому он будет соответствовать `'/messages/<int:id>'` и запускать метод `MessageResource.patch`, то есть метод **patch** для класса **MessageResource**. Если экземпляр сообщения с указанным идентификатором существует, код извлечет значения для ключей **print\_times** и **print\_once** в словаре запросов, обновит экземпляр сообщения и проверит его.

Если обновленный экземпляр **Message** действителен, код сохранит изменения в базе данных, а вызов метода вернет код состояния HTTP **200 OK** и недавно обновленный экземпляр **Message**, сериализованный в **JSON** в теле ответа. В следующих строках показан пример ответа:

```http
HTTP/1.0 200 OK
Content-Length: 368
Content-Type: application/json
Date: Tue, 09 Aug 2016 22:38:39 GMT
Server: Werkzeug/0.11.10 Python/3.5.1
{
    "category": {
        "id": 1,
        "name": "Information",
        "url": "http://localhost:5000/api/categories/1"
    },
    "creation_date": "2016-08-08T12:18:43.260474+00:00",
    "duration": 5,
    "id": 1,
    "message": "Checking temperature sensor",
    "printed_once": true,
    "printed_times": 1,
    "url": "http://localhost:5000/api/messages/1"
}
```

Мы можем запустить команды, описанные в [Главе 6](glava-6.-rabota-s-modelyami-sqlalchemy-i-api-s-giperssylkami-v-flask.md), "_Работа с моделями, SQLAlchemy и API-интерфейсами с гиперссылками во Flask_", чтобы проверить содержимое таблиц, созданных миграциями в базе данных **PostgreSQL**. Мы заметим, что значения **print\_times** и **print\_once** были обновлены для строки в таблице сообщений. На следующем снимке экрана показано содержимое обновленной строки таблицы сообщений в базе данных PostgreSQL после выполнения HTTP-запроса. На скриншоте показаны результаты выполнения следующего SQL-запроса: `SELECT * FROM message WHERE id = 1`:

![](../../.gitbook/assets/db\_3.png)

## Кодирование универсального класса пагинации

В нашей базе данных есть несколько строк для каждой из таблиц, которые сохраняют определенные нами модели. Однако после того, как мы начнем работать с нашим API в реальной производственной среде, у нас будут сотни сообщений, а значит, нам придется иметь дело с большими наборами результатов. Таким образом, мы создадим общий класс разбиения на страницы и будем использовать его, чтобы легко указать, как мы хотим, чтобы большие наборы результатов были разделены на отдельные страницы данных.

Сначала мы составим и отправим HTTP-запросы для создания 9 сообщений, принадлежащих к одной из созданных нами категорий: **Information**. Таким образом, в базе данных будет храниться 12 сообщений. У нас было 3 сообщения и мы добавляем еще 9.

```bash
http POST :5000/api/messages/ message='Initializing light controller'
duration=25 category="Information"
http POST :5000/api/messages/ message='Initializing light sensor' duration=20
category="Information"
http POST :5000/api/messages/ message='Checking pressure sensor' duration=18
category="Information"
http POST :5000/api/messages/ message='Checking gas sensor' duration=14
category="Information"
http POST :5000/api/messages/ message='Setting ADC resolution' duration=22
category="Information"
http POST :5000/api/messages/ message='Setting sample rate' duration=15
category="Information"
http POST :5000/api/messages/ message='Initializing pressure sensor'
duration=18 category="Information"
http POST :5000/api/messages/ message='Initializing gas sensor' duration=16
category="Information"
http POST :5000/api/messages/ message='Initializing proximity sensor'
duration=5 category="Information"
```

Ниже приведены эквивалентные команды **curl**:

```bash
curl -iX POST -H "Content-Type: application/json" -d '{"message":"
Initializing light controller", "duration":25, "category": "Information"}'
:5000/api/messages/
curl -iX POST -H "Content-Type: application/json" -d '{"message":"Initializing
light sensor", "duration":20, "category": "Information"}' :5000/api/messages/
curl -iX POST -H "Content-Type: application/json" -d '{"message":"Checking
pressure sensor", "duration":18, "category": "Information"}'
:5000/api/messages/
curl -iX POST -H "Content-Type: application/json" -d '{"message":"Checking gas
sensor", "duration":14, "category": "Information"}' :5000/api/messages/
curl -iX POST -H "Content-Type: application/json" -d '{"message":"Setting ADC
resolution", "duration":22, "category": "Information"}' :5000/api/messages/
curl -iX POST -H "Content-Type: application/json" -d '{"message":"Setting
sample rate", "duration":15, "category": "Information"}' :5000/api/messages/
curl -iX POST -H "Content-Type: application/json" -d '{"message":"Initializing
pressure sensor", "duration":18, "category": "Information"}'
:5000/api/messages/
curl -iX POST -H "Content-Type: application/json" -d '{"message":"Initializing
gas sensor", "duration":16, "category": "Information"}' :5000/api/messages/
curl -iX POST -H "Content-Type: application/json" -d '{"message":"Initializing
proximity sensor", "duration":5, "category": "Information"}'
:5000/api/messages/
```

Предыдущие команды составят и отправят девять HTTP-запросов **POST** с указанными парами ключ-значение **JSON**. В запросе указывается `/api/messages/`, поэтому он будет соответствовать `'/messages/'` и запускать метод `MessageListResource.post`, то есть метод **post** для класса **MessageListResource**.

Теперь у нас есть 12 сообщений в нашей базе данных. Однако мы не хотим получать 12 сообщений при составлении и отправке HTTP-запроса **GET** в `/api/messages/`. Мы создадим настраиваемый общий класс разбивки на страницы, чтобы включить максимум 5 ресурсов на каждой отдельной странице данных.

Откройте файл `api/config.py` и добавьте следующие строки, объявляющие две переменные, которые настраивают глобальные параметры разбиения на страницы. Файл кода для примера включен в папку `restful_python_chapter_07_01`:

```python
PAGINATION_PAGE_SIZE = 5
PAGINATION_PAGE_ARGUMENT_NAME = 'page'
```

Значение переменной **PAGINATION\_PAGE\_SIZE** указывает глобальную настройку со значением по умолчанию для размера страницы, также называемым пределом. Значение для **PAGINATION\_PAGE\_ARGUMENT\_NAME** указывает глобальную настройку со значением по умолчанию для имени аргумента, которое мы будем использовать в наших запросах, чтобы указать номер страницы, которую мы хотим получить.

Создайте новый файл **helpers.py** в папке **api**. В следующих строках показан код, создающий новый класс **PaginationHelper**. Файл кода для примера включен в папку `restful_python_chapter_07_01`:

```python
from flask import url_for
from flask import current_app

class PaginationHelper():
    def __init__(self, request, query, resource_for_url, key_name, schema):
        self.request = request
        self.query = query
        self.resource_for_url = resource_for_url
        self.key_name = key_name
        self.schema = schema
        self.results_per_page = current_app.config['PAGINATION_PAGE_SIZE']
        self.page_argument_name = current_app.config['PAGINATION_PAGE_ARGUMENT_NAME']

    def paginate_query(self):
        # Если номер страницы не указан, мы предполагаем,
        # что запрос требует страницу № 1.
        page_number = self.request.args.get(self.page_argument_name, 1, type=int)
        paginated_objects = self.query.paginate(
            page_number,
            per_page=self.results_per_page,
            error_out=False)
        objects = paginated_objects.items
        if paginated_objects.has_prev:
            previous_page_url = url_for(
                self.resource_for_url,
                page=page_number-1,
                _external=True)
        else:
            previous_page_url = None
        if paginated_objects.has_next:
            next_page_url = url_for(
                self.resource_for_url,
                page=page_number+1,
                _external=True)
        else:
            next_page_url = None
        dumped_objects = self.schema.dump(objects, many=True).data
        return ({
            self.key_name: dumped_objects,
            'previous': previous_page_url,
            'next': next_page_url,
            'count': paginated_objects.total
        })
```

В классе **PaginationHelper** объявляется конструктор, то есть метод **\_\_init\_\_**, который получил множество аргументов и использует их для инициализации атрибутов с такими именами:

* **request**: Объект запроса Flask, который позволит методу **paginate\_query** получить значение номера страницы, указанное в HTTP-запросе.
* **query**: запрос SQLAlchemy, который метод **paginate\_query** должен разбить на страницы.
* **resource\_for\_url**: строка с именем ресурса, которую метод **paginate\_query** будет использовать для создания полных URL-адресов для предыдущей и следующей страниц.
* **key\_name**: строка с именем ключа, которую метод **paginate\_query** будет использовать для возврата сериализованных объектов.
* **schema**: подкласс схемы Flask-Marshmallow, который метод **paginate\_query** должен использовать для сериализации объектов.

Кроме того, конструктор считывает и сохраняет значения переменных конфигурации, которые мы добавили в файл **config.py** в атрибутах **results\_per\_page** и **page\_argument\_name**.

Класс объявляет метод **paginate\_query**. Сначала код извлекает номер страницы, указанный в запросе, и сохраняет его в переменной **page\_number**. Если номер страницы не указан, код предполагает, что для запроса требуется первая страница. Затем код вызывает метод `self.query.paginate` для получения номера страницы, указанного в параметре **page\_number** разбивки на страницы результатов объектов из базы данных, с количеством результатов на странице, указанным значением атрибута `self.results_per_page`. Следующая строка сохраняет элементы с разбивкой на страницы из атрибута `paginated_object.items` в переменной объектов.

Если значение атрибута `paginated_objects.has_prev` равно `True`, это означает, что доступна предыдущая страница. В этом случае код вызывает функцию `flask.url_for` для создания **полного** URL-адреса предыдущей страницы со значением атрибута `self.resource_for_url`. Аргумент **\_external** имеет значение `True`, потому что мы хотим указать полный URL-адрес.

Если значение атрибута `paginated_objects.has_next` равно `True`, это означает, что доступна следующая страница. В этом случае код вызывает функцию `flask.url_for` для создания **полного** URL-адреса следующей страницы со значением атрибута `self.resource_for_url`.

Затем код вызывает метод `self.schema.dump` для сериализации частичных результатов, ранее сохраненных в объектной переменной, с аргументом **many**, для которого задано значение `True`. Переменная **dumped\_objects** сохраняет ссылку на атрибут данных результатов, возвращаемых вызовом метода дампа.

Наконец, метод возвращает словарь со следующими парами ключ-значение:

* **self.key\_name**: сериализованные частичные результаты, сохраненные в переменной **dumped\_objects**.
* **`'previous'`**: полный URL-адрес предыдущей страницы, сохраненный в переменной **previous\_page\_url**.
* **`'next'`**: полный URL-адрес следующей страницы, сохраненный в переменной **next\_page\_url**.
* **`'count'`**: общее количество объектов, доступных в полном наборе результатов, полученном из атрибута **paginated\_objects.total**.

## Добавление функций пагинации

Откройте файл `api/views.py` и замените код метода `MessageListResource.get` выделенными строками в следующем листинге. Кроме того, убедитесь, что вы добавили оператор **import**. Файл кода для примера включен в папку `restful_python_chapter_07_01`:

```python
from helpers import PaginationHelper

class MessageListResource(Resource):
    def get(self):
        pagination_helper = PaginationHelper(
            request,
            query=Message.query,
            resource_for_url='api.messagelistresource',
            key_name='results',
            schema=message_schema)
        result = pagination_helper.paginate_query()
        return result
```

Новый код для метода **get** создает экземпляр ранее объясненного класса **PaginationHelper** с именем **pagination\_helper** с объектом запроса в качестве первого аргумента. Именованные аргументы определяют запрос **request**, **resource\_for\_url**, **key\_name** и схему **schema**, которые должен использовать экземпляр **PaginationHelper** для предоставления результата запроса с разбивкой на страницы.

Следующая строка вызывает метод `pagination_helper.paginate_query`, который вернет результаты запроса с разбивкой на страницы с указанным в запросе номером страницы. Наконец, метод возвращает результаты вызова этого метода, которые включают ранее объясненный словарь. В этом случае разбитый на страницы набор результатов с сообщениями будет отображаться как значение ключа `'results'`, указанного в аргументе **key\_name**.

Теперь мы создадим и отправим HTTP-запрос для получения всех сообщений, в частности, метод HTTP **GET** для `/api/messages/`.

```bash
http :5000/api/messages/
```

Ниже приведена эквивалентная команда **curl**:

```bash
curl -iX GET :5000/api/messages/
```

Новый код для метода `MessageListResource.get` будет работать с разбиением на страницы, и в результате мы получим первые 5 сообщений (results key), общее количество сообщений для запроса (count key) и ссылку на следующее (next key) и предыдущие (previous key) страницы. В этом случае набор результатов — это первая страница, поэтому ссылка на предыдущую страницу (previous key) пуста. Мы получим код состояния **200 OK** в заголовке ответа и 5 сообщений в массиве результатов:

```http
HTTP/1.0 200 OK
Content-Length: 2521
Content-Type: application/json
Date: Wed, 10 Aug 2016 18:26:44 GMT
Server: Werkzeug/0.11.10 Python/3.5.1
{
    "count": 12,
    "results": [
        {
            "category": {
                "id": 1,
                "name": "Information",
                "url": "http://localhost:5000/api/categories/1"
            },
            "creation_date": "2016-08-08T12:27:30.124511+00:00",
            "duration": 8,
            "id": 2,
            "message": "Checking light sensor",
            "printed_once": false,
            "printed_times": 0,
            "url": "http://localhost:5000/api/messages/2"
        },
        {
            "category": {
                "id": 3,
                "name": "Error",
                "url": "http://localhost:5000/api/categories/3"
            },
            "creation_date": "2016-08-08T14:20:22.103752+00:00",
            "duration": 10,
            "id": 3,
            "message": "Temperature sensor error",
            "printed_once": false,
            "printed_times": 0,
            "url": "http://localhost:5000/api/messages/3"
        },
        {
            "category": {
                "id": 1,
                "name": "Information",
                "url": "http://localhost:5000/api/categories/1"
            },
            "creation_date": "2016-08-08T12:18:43.260474+00:00",
            "duration": 5,
            "id": 1,
            "message": "Checking temperature sensor",
            "printed_once": true,
            "printed_times": 1,
            "url": "http://localhost:5000/api/messages/1"
        },
        {
            "category": {
                "id": 1,
                "name": "Information",
                "url": "http://localhost:5000/api/categories/1"
            },
            "creation_date": "2016-08-09T20:18:26.648071+00:00",
            "duration": 25,
            "id": 4,
            "message": "Initializing light controller",
            "printed_once": false,
            "printed_times": 0,
            "url": "http://localhost:5000/api/messages/4"
        },
        {
            "category": {
                "id": 1,
                "name": "Information",
                "url": "http://localhost:5000/api/categories/1"
            },
            "creation_date": "2016-08-09T20:19:16.174807+00:00",
            "duration": 20,
            "id": 5,
            "message": "Initializing light sensor",
            "printed_once": false,
            "printed_times": 0,
            "url": "http://localhost:5000/api/messages/5"
        }
    ],
    "next": "http://localhost:5000/api/messages/?page=2",
    "previous": null
}
```

В предыдущем HTTP-запросе мы не указали никакого значения для параметра страницы, поэтому метод **paginate\_query** в классе **PaginationHelper** запрашивает первую страницу для запроса с разбивкой на страницы. Если мы составим и отправим следующий HTTP-запрос для получения первой страницы всех сообщений, указав 1 для значения страницы, API предоставит те же результаты, что и раньше:

```bash
http ':5000/api/messages/?page=1'
```

Ниже приведена эквивалентная команда **curl**:

```bash
curl -iX GET ':5000/api/messages/?page=1'
```

{% hint style="info" %}
**СОВЕТ**

Код в классе **PaginationHelper** считает, что первой страницей является страница с номером **1**. Таким образом, мы не работаем с нумерацией страниц, начинающейся с нуля.
{% endhint %}

Теперь мы создадим и отправим HTTP-запрос для получения следующей страницы, то есть второй страницы для сообщений, в частности, метод HTTP **GET** для `/api/messages/` со значением страницы, равным **2**. Помните, что значение для следующего ключа, возвращенное в теле JSON предыдущего результата, предоставляет нам полный URL-адрес следующей страницы:

```bash
http ':5000/api/messages/?page=2'
```

Ниже приведена эквивалентная команда **curl**:

```bash
curl -iX GET ':5000/api/messages/?page=2'
```

Результат предоставит нам второй набор из пяти ресурсов сообщений (results key), общее количество сообщений для запроса (count key), ссылку на следующую (next key) и предыдущую (previous key) страницы. В данном случае набор результатов — это вторая страница, поэтому ссылка на предыдущую страницу (previous key) — `http://localhost:5000/api/messages/?page=1`. Мы получим код состояния **200 OK** в заголовке ответа и 5 сообщений в массиве результатов.

```http
HTTP/1.0 200 OK
Content-Length: 2557
Content-Type: application/json
Date: Wed, 10 Aug 2016 19:51:50 GMT
Server: Werkzeug/0.11.10 Python/3.5.1
{
    "count": 12,
    "next": "http://localhost:5000/api/messages/?page=3",
    "previous": "http://localhost:5000/api/messages/?page=1",
    "results": [
        {
            "category": {
                "id": 1,
                "name": "Information",
                "url": "http://localhost:5000/api/categories/1"
            },
            "creation_date": "2016-08-09T20:19:22.335600+00:00",
            "duration": 18,
            "id": 6,
            "message": "Checking pressure sensor",
            "printed_once": false,
            "printed_times": 0,
            "url": "http://localhost:5000/api/messages/6"
        },
        {
            "category": {
                "id": 1,
                "name": "Information",
                "url": "http://localhost:5000/api/categories/1"
            },
            "creation_date": "2016-08-09T20:19:26.189009+00:00",
            "duration": 14,
            "id": 7,
            "message": "Checking gas sensor",
            "printed_once": false,
            "printed_times": 0,
            "url": "http://localhost:5000/api/messages/7"
        },
        {
            "category": {
                "id": 1,
                "name": "Information",
                "url": "http://localhost:5000/api/categories/1"
            },
            "creation_date": "2016-08-09T20:19:29.854576+00:00",
            "duration": 22,
            "id": 8,
            "message": "Setting ADC resolution",
            "printed_once": false,
            "printed_times": 0,
            "url": "http://localhost:5000/api/messages/8"
        },
        {
            "category": {
                "id": 1,
                "name": "Information",
                "url": "http://localhost:5000/api/categories/1"
            },
            "creation_date": "2016-08-09T20:19:33.838977+00:00",
            "duration": 15,
            "id": 9,
            "message": "Setting sample rate",
            "printed_once": false,
            "printed_times": 0,
            "url": "http://localhost:5000/api/messages/9"
        },
        {
            "category": {
                "id": 1,
                "name": "Information",
                "url": "http://localhost:5000/api/categories/1"
            },
            "creation_date": "2016-08-09T20:19:37.830843+00:00",
            "duration": 18,
            "id": 10,
            "message": "Initializing pressure sensor",
            "printed_once": false,
            "printed_times": 0,
            "url": "http://localhost:5000/api/messages/10"
        }
    ]
}
```

Наконец, мы создадим и отправим HTTP-запрос для получения последней страницы, то есть третьей страницы для сообщений, в частности, метод HTTP **GET** для `/api/messages/` со значением страницы равным **3**. Помните, что значение для следующего ключа, возвращенное в теле JSON предыдущего результата, предоставляет нам URL-адрес следующей страницы:

```bash
http ':5000/api/messages/?page=3'
```

Ниже приведена эквивалентная команда **curl**:

```bash
curl -iX GET ':5000/api/messages/?page=3'
```

Результат предоставит нам последний набор с двумя ресурсами сообщений (results key), общее количество сообщений для запроса (count key), ссылку на следующую (next key) и предыдущую (previous key) страницы. В этом случае набор результатов является последней страницей, поэтому ссылка на следующую страницу (previous key) пуста. Мы получим код состояния **200 OK** в заголовке ответа и 2 сообщения в массиве результатов:

```http
HTTP/1.0 200 OK
Content-Length: 1090
Content-Type: application/json
Date: Wed, 10 Aug 2016 20:02:00 GMT
Server: Werkzeug/0.11.10 Python/3.5.1
{
    "count": 12,
    "next": null,
    "previous": "http://localhost:5000/api/messages/?page=2",
    "results": [
        {
            "category": {
                "id": 1,
                "name": "Information",
                "url": "http://localhost:5000/api/categories/1"
            },
            "creation_date": "2016-08-09T20:19:41.645628+00:00",
            "duration": 16,
            "id": 11,
            "message": "Initializing gas sensor",
            "printed_once": false,
            "printed_times": 0,
            "url": "http://localhost:5000/api/messages/11"
        },
        {
            "category": {
                "id": 1,
                "name": "Information",
                "url": "http://localhost:5000/api/categories/1"
            },
            "creation_date": "2016-08-09T20:19:45.304391+00:00",
            "duration": 5,
            "id": 12,
            "message": "Initializing proximity sensor",
            "printed_once": false,
            "printed_times": 0,
            "url": "http://localhost:5000/api/messages/12"
        }
    ]
}
```

## Понимание шагов по добавлению проверки подлинности и разрешений

Наша текущая версия API обрабатывает все входящие запросы без какой-либо аутентификации. Мы будем использовать расширение Flask и другие пакеты, чтобы использовать схему аутентификации HTTP для идентификации пользователя, отправившего запрос, или токена, подписавшего запрос. Затем мы будем использовать эти учетные данные для применения разрешений, которые будут определять, должен ли запрос быть разрешен или нет. К сожалению, ни Flask, ни Flask-RESTful не предоставляют инфраструктуру аутентификации, которую мы можем легко подключить и настроить. Таким образом, нам придется написать код для выполнения многих задач, связанных с аутентификацией и разрешениями.

Мы хотим иметь возможность создавать нового пользователя без какой-либо аутентификации. Однако все остальные вызовы API будут доступны только для аутентифицированных пользователей.

Во-первых, мы установим расширение Flask, чтобы нам было проще работать с HTTP-аутентификацией, **Flask-HTTPAuth**, и пакет, позволяющий нам хешировать пароль и проверять, действителен ли предоставленный пароль, **passlib**.

Мы создадим новую модель **User**, которая будет представлять пользователя. Модель предоставит методы, позволяющие нам хешировать пароль и проверять, является ли пароль, предоставленный пользователю, действительным или нет. Мы создадим класс **UserSchema**, чтобы указать, как мы хотим сериализовать и десериализовать пользователя.

Затем мы настроим расширение Flask для работы с нашей моделью **User** для проверки паролей и установки аутентифицированного пользователя, связанного с запросом. Мы внесем изменения в существующие ресурсы, чтобы требовать аутентификацию, и мы добавим новые ресурсы, чтобы позволить нам извлекать существующих пользователей и создавать новых. Наконец, мы настроим маршруты для ресурсов, связанных с пользователями.

После того, как мы выполнили ранее упомянутые задачи, мы запустим миграции, чтобы создать новую таблицу, которая сохранит пользователей в базе данных. Затем мы составим и отправим HTTP-запросы, чтобы понять, как аутентификация и разрешения работают с нашей новой версией API.

Убедитесь, что вы вышли из сервера разработки **Flask**. Помните, что вам просто нужно нажать **Ctrl + C** в терминале или окне командной строки, в котором он запущен. Пришло время запустить множество команд, которые будут одинаковыми для macOS, Linux или Windows. Мы можем установить все необходимые пакеты с помощью **pip** с помощью одной команды. Однако мы запустим две независимые команды, чтобы упростить обнаружение любых проблем в случае сбоя конкретной установки.

Теперь мы должны запустить следующую команду, чтобы установить **Flask-HTTPAuth** с **pip**. Этот пакет упрощает добавление базовой HTTP-аутентификации в любое приложение **Flask**:

```bash
pip install Flask-HTTPAuth
```

Последние строки вывода будут означать, что пакет **Flask-HTTPAuth** был успешно установлен:

```bash
Installing collected packages: Flask-HTTPAuth
  Running setup.py install for Flask-HTTPAuth
Successfully installed Flask-HTTPAuth-3.2.1
```

Выполните следующую команду, чтобы установить **passlib** с **pip**. Этот популярный пакет предоставляет комплексную структуру хэширования паролей, которая поддерживает более 30 схем. Мы определенно не хотим писать свой собственный подверженный ошибкам и, возможно, очень небезопасный код хэширования паролей, и поэтому мы воспользуемся библиотекой, которая предоставляет следующие услуги:

```bash
pip install passlib
```

Последние строки вывода укажут, что пакет **passlib** был успешно установлен:

```bash
Installing collected packages: passlib
Successfully installed passlib-1.6.5
```

## Добавление модели пользователя

Теперь мы создадим модель, которую будем использовать для представления и сохранения пользователя. Откройте файл `api/models.py` и добавьте следующие строки после объявления класса **AddUpdateDelete**. Убедитесь, что вы добавили операторы импорта. Файл кода для примера включен в папку `restful_python_chapter_07_02`:

```python
from passlib.apps import custom_app_context as password_context
import re

class User(db.Model, AddUpdateDelete):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), unique=True, nullable=False)
    # Я сохраняю хешированный пароль
    hashed_password = db.Column(db.String(120), nullable=False)
    creation_date = db.Column(
        db.TIMESTAMP,
        server_default=db.func.current_timestamp(),
        nullable=False)

    def verify_password(self, password):
        return password_context.verify(password, self.hashed_password)

    def check_password_strength_and_hash_if_ok(self, password):
        if len(password) < 8:
            return 'The password is too short', False
        if len(password) > 32:
            return 'The password is too long', False
        if re.search(r'[A-Z]', password) is None:
            return 'The password must include at least one uppercase letter', False
        if re.search(r'[a-z]', password) is None:
            return 'The password must include at least one lowercase letter', False
        if re.search(r'\d', password) is None:
            return 'The password must include at least one number', False
        if re.search(r"[ !#$%&'()*+,-./[\\\]^_`{|}~"+r'"]', password) is None:
            return 'The password must include at least one symbol', False
        self.hashed_password = password_context.encrypt(password)
        return '', True

    def __init__(self, name):
        self.name = name
```

Код объявляет модель **User**, в частности, подклассы классов **db.Model** и **AddUpdateDelete**. Мы указали типы полей, максимальную длину и значения по умолчанию для следующих трех атрибутов: идентификатора **id**, имени **name**, **hashed\_password** и даты создания **creation\_date**. Эти атрибуты представляют поля без какой-либо связи, и поэтому они являются экземплярами класса **db.Column**. Модель объявляет атрибут **id** и указывает значение `True` для аргумента **primary\_key**, чтобы указать, что это первичный ключ. **SQLAlchemy** будет использовать данные для создания необходимой таблицы в базе данных **PostgreSQL**.

Класс **User** объявляет следующие методы:

* **check\_password\_strength\_and\_hash\_if\_ok**: этот метод использует модуль **re**, который обеспечивает операции сопоставления регулярных выражений, чтобы проверить, соответствует ли пароль, полученный в качестве аргумента, многим качественным требованиям. Код требует, чтобы пароль был длиннее восьми символов, но не более 32 символов. Пароль должен содержать как минимум одну заглавную букву, одну строчную букву, одну цифру и один символ. Код проверяет результаты многих вызовов метода `re.search`, чтобы определить, соответствует ли полученный пароль каждому требованию. Если какое-либо из требований не выполняется, код возвращает кортеж с сообщением об ошибке и значением `False`. В противном случае код вызывает метод шифрования для экземпляра `passlib.apps.custom_app_context`, импортированного как **password\_context**, с полученным паролем в качестве аргумента. Метод шифрования выбирает достаточно надежную схему на основе платформы с настройками по умолчанию для выбора раундов, а код сохраняет хешированный пароль в атрибуте **hash\_password**. Наконец, код возвращает кортеж с пустой строкой и значением `True`, что указывает на то, что пароль соответствует качественным требованиям и был хеширован.

{% hint style="info" %}
**СОВЕТ**

По умолчанию библиотека **passlib** будет использовать схему **SHA-512** для 64-битных платформ и **SHA-256** для 32-битных платформ. Кроме того, минимальное количество раундов будет установлено на уровне 535 000. В этом примере мы будем использовать значения конфигурации по умолчанию. Однако вы должны принять во внимание, что эти значения могут потребовать слишком много времени для обработки каждого запроса, который должен проверять пароль. Вы обязательно должны выбрать наиболее подходящий алгоритм и количество раундов, исходя из ваших требований безопасности.
{% endhint %}

* **verify\_password**: этот метод вызывает метод **verify** для экземпляра `passlib.apps.custom_app_context`, импортированный как **password\_context**, с полученным паролем и сохраненным хешированным паролем для пользователя, `self.hashed_password`, в качестве аргументов. Метод **verify** хэширует полученный пароль и возвращает значение `True`, только если полученный хешированный пароль совпадает с сохраненным хешированным паролем. Мы никогда не восстанавливаем сохраненный пароль в исходное состояние. Мы просто сравниваем хешированные значения.

Модель объявляет конструктор, то есть метод **\_\_init\_\_**. Этот конструктор получает имя пользователя в аргументе **name** и сохраняет его в атрибуте с тем же именем.

## Создание схем для проверки, сериализации и десериализации пользователей

Теперь мы создадим схему **Flask-Marshmallow**, которую будем использовать для проверки, сериализации и десериализации ранее объявленной модели **User**. Откройте файл `api/models.py` и добавьте следующий код после существующих строк. Файл кода для примера включен в папку `restful_python_chapter_07_02`:

```python
class UserSchema(ma.Schema):
    id = fields.Integer(dump_only=True)
    name = fields.String(required=True, validate=validate.Length(3))
    url = ma.URLFor('api.userresource', id='<id>', _external=True)
```

Код объявляет схему **UserSchema**, в частности, подкласс класса **ma.Schema**. Помните, что предыдущий код, который мы написали для файла `api/models.py`, создал экземпляр `flask_marshmallow.Mashmallow` с именем **ma**.

Мы объявляем атрибуты, представляющие поля, как экземпляры соответствующего класса, объявленного в модуле `marshmallow.fields`. Класс **UserSchema** объявляет атрибут **name** как экземпляр `fields.String`. Обязательный аргумент **required** имеет значение `True`, чтобы указать, что поле не может быть пустой строкой. Аргументу проверки присвоено значение `validate.Length(3)`, чтобы указать, что поле должно иметь минимальную длину 3 символа.

Проверка пароля не включена в схему. Мы будем использовать метод **check\_password\_strength\_and\_hash\_if\_ok**, определенный в классе **User**, для проверки пароля.

## Добавление аутентификации к ресурсам

Мы настроим расширение **Flask-HTTPAuth** для работы с нашей моделью **User** для проверки паролей и установки аутентифицированного пользователя, связанного с запросом. Мы объявим пользовательскую функцию, которую это расширение будет использовать в качестве обратного вызова для проверки пароля. Мы создадим новый базовый класс для наших ресурсов, который потребует аутентификации. Откройте файл `api/views.py` и добавьте следующий код после последней строки, в которой используется оператор **import**, и перед строками, в которых объявляется экземпляр **Blueprint**. Файл кода для примера включен в папку `restful_python_chapter_07_02`:

```python
from flask_httpauth import HTTPBasicAuth
from flask import g
from models import User, UserSchema

auth = HTTPBasicAuth()

@auth.verify_password
def verify_user_password(name, password):
    user = User.query.filter_by(name=name).first()
    if not user or not user.verify_password(password):
        return False
    g.user = user
    return True

class AuthRequiredResource(Resource):
    method_decorators = [auth.login_required]
```

Сначала мы создаем экземпляр класса `flask_httpauth.HTTPBasicAuth` с именем **auth**. Затем мы объявляем функцию **verify\_user\_password**, которая получает в качестве аргументов имя и пароль. Функция использует декоратор `@auth.verify_password`, чтобы сделать эту функцию обратным вызовом, который **Flask-HTTPAuth** будет использовать для проверки пароля для конкретного пользователя. Функция извлекает пользователя, чье имя **name** совпадает с именем, указанным в аргументе, и сохраняет его ссылку в переменной **user**. Если пользователь найден, код проверяет результаты метода `user.verify_password` с полученным паролем в качестве аргумента.

Если либо пользователь не найден, либо вызов `user.verify_password` возвращает значение `False`, функция возвращает значение `False`, и аутентификация завершается ошибкой. Если вызов `user.verify_password` возвращает `True`, функция сохраняет аутентифицированный экземпляр пользователя в атрибуте **user** для объекта `flask.g`.

{% hint style="info" %}
**СОВЕТ**

Объект `flask.g` — это прокси, который позволяет нам хранить на нем все, чем мы хотим поделиться, только для одного запроса. Атрибут пользователя **user**, который мы добавили в объект `flask.g`, будет действителен только для активного запроса и будет возвращать разные значения для каждого другого запроса. Таким образом, можно использовать `flask.g.user` в другой функции или методе, вызываемом во время запроса, чтобы получить доступ к сведениям об аутентифицированном пользователе для запроса.
{% endhint %}

Наконец, мы объявили класс **AuthRequiredResource** как подкласс `flask_restful.Resource`. Мы только что указали `auth.login_required` как один из членов списка, который мы назначаем свойству **method\_decorators**, унаследованному от базового класса. Таким образом, _**ко всем методам**_, объявленным в ресурсе, который использует новый класс **AuthRequiredResource** в качестве своего суперкласса, будет применяться декоратор `auth.login_required`, и, следовательно, любой метод, вызываемый для ресурса, потребует аутентификации.

Теперь мы заменим базовый класс для существующих классов ресурсов, чтобы они наследуются от **AuthRequiredResource** вместо **Resource**. Мы хотим, чтобы любые запросы, которые извлекают или изменяют категории и сообщения, были аутентифицированы.

В следующих строках показаны объявления для четырех классов ресурсов:

```python
class MessageResource(Resource):
class MessageListResource(Resource):
class CategoryResource(Resource):
class CategoryListResource(Resource):
```

Откройте файл `api/views.py` и замените **Resource** на **AuthRequiredResource** в ранее показанных четырех строках, объявляющих классы ресурсов. В следующих строках показан новый код для каждого объявления класса ресурсов:

```python
class MessageResource(AuthRequiredResource):
class MessageListResource(AuthRequiredResource):
class CategoryResource(AuthRequiredResource):
class CategoryListResource(AuthRequiredResource):
```

## Создание классов ресурсов для обработки пользователей

Мы просто хотим иметь возможность создавать пользователей и использовать их для аутентификации запросов. Таким образом, мы просто сосредоточимся на создании классов ресурсов с помощью всего нескольких методов. Мы не будем создавать полную систему управления пользователями.

Мы создадим классы ресурсов, которые представляют пользователя и набор пользователей. Сначала мы создадим класс **UserResource**, который будем использовать для представления пользовательского ресурса. Откройте файл `api/views.py` и добавьте следующие строки после строки, создающей экземпляр **Api**. Файл кода для примера включен в папку `restful_python_chapter_07_02`:

```python
class UserResource(AuthRequiredResource):
  def get(self, id):
    user = User.query.get_or_404(id)
    result = user_schema.dump(user).data
    return result
```

Класс **UserResource** является подклассом ранее закодированного **AuthRequiredResource** и объявляет методы **get**, которые будут вызываться, когда HTTP-метод с тем же именем поступает в качестве запроса на представленный ресурс. Метод получает идентификатор пользователя, который необходимо получить в аргументе идентификатора. Код вызывает метод `User.query.get_or_404`, чтобы вернуть статус HTTP **404 Not Found**, если в базовой базе данных нет пользователя с запрошенным идентификатором. Если пользователь существует, код вызывает метод `user_schema.dump` с полученным пользователем в качестве аргумента, чтобы использовать экземпляр **UserSchema** для сериализации экземпляра **User**, идентификатор которого соответствует указанному идентификатору. Метод дампа принимает экземпляр **User** и применяет фильтрацию полей и форматирование вывода, указанные в классе **UserSchema**. Фильтрация полей указывает, что мы не хотим сериализовать хешированный пароль. Код возвращает атрибут данных результата, возвращенного методом дампа, то есть сериализованное сообщение в формате **JSON** в качестве тела с кодом состояния HTTP **200 OK** по умолчанию.

Теперь мы создадим класс **UserListResource**, который будем использовать для представления коллекции пользователей. Откройте файл `api/views.py` и добавьте следующие строки после кода, создающего класс **UserResource**. Файл кода для примера включен в папку `restful_python_chapter_07_02`:

```python
class UserListResource(Resource):
    @auth.login_required
    def get(self):
        pagination_helper = PaginationHelper(
            request,
            query=User.query,
            resource_for_url='api.userlistresource',
            key_name='results',
            schema=user_schema)
        result = pagination_helper.paginate_query()
        return result

    def post(self):
        request_dict = request.get_json()
        if not request_dict:
            response = {'user': 'No input data provided'}
            return response, status.HTTP_400_BAD_REQUEST
        errors = user_schema.validate(request_dict)
        if errors:
            return errors, status.HTTP_400_BAD_REQUEST
        name = request_dict['name']
        existing_user = User.query.filter_by(name=name).first()
        if existing_user is not None:
            response = {'user': 'An user with the same name already exists'}
            return response, status.HTTP_400_BAD_REQUEST
        try:
            user = User(name=name)
            error_message, password_ok = \
            user.check_password_strength_and_hash_if_ok(request_dict['password'])
            if password_ok:
                user.add(user)
                query = User.query.get(user.id)
                result = user_schema.dump(query).data
                return result, status.HTTP_201_CREATED
            else:
                return {"error": error_message}, status.HTTP_400_BAD_REQUEST
        except SQLAlchemyError as e:
            db.session.rollback()
            resp = {"error": str(e)}
            return resp, status.HTTP_400_BAD_REQUE
```

Класс **UserListResource** является подклассом `flask_restful.Resource`, потому что мы не хотим, чтобы все методы требовали аутентификации. Мы хотим иметь возможность создавать нового пользователя без аутентификации, поэтому применяем декоратор `@auth.login_required` только для метода **get**. Метод **post** не требует аутентификации. В классе объявляются следующие два метода, которые будут вызываться при поступлении одноименного HTTP-метода в качестве запроса на представляемый ресурс:

* **get**: этот метод возвращает список со всеми экземплярами **User**, сохраненными в базе данных. Во-первых, код вызывает метод `User.query.all` для извлечения всех экземпляров **User**, сохраненных в базе данных. Затем код вызывает метод `user_schema.dump` с извлеченными сообщениями и аргументом **many**, для которого задано значение `True`, чтобы сериализовать итерируемую коллекцию объектов. Метод **dump** возьмет каждый экземпляр пользователя, полученный из базы данных, и применит фильтрацию полей и форматирование вывода, указанные в классе **UserSchema**. Код возвращает атрибут **data** результата, возвращенного методом **dump**, то есть сериализованные сообщения в формате **JSON** в качестве тела с кодом состояния HTTP **200 OK** по умолчанию.
* **post**: этот метод извлекает пары ключ-значение, полученные в теле **JSON**, создает новый экземпляр пользователя **User** и сохраняет его в базе данных. Во-первых, код вызывает метод `request.get_json` для получения пар ключ-значение, полученных в качестве аргументов запроса. Затем код вызывает метод `user_schema.validate` для проверки нового пользователя, созданного с помощью полученных пар ключ-значение. В этом случае вызов этого метода просто проверит поле **name** для пользователя. В случае ошибок проверки код возвращает ошибки проверки и статус HTTP **400 Bad Request**. Если проверка прошла успешно, код проверяет, существует ли уже пользователь с таким именем в базе данных, или нет, чтобы вернуть соответствующую ошибку для поля, которое должно быть уникальным. Если имя пользователя уникально, код создает нового пользователя с указанным именем и вызывает его метод **check\_password\_strength\_and\_hash\_if\_ok**. Если предоставленный пароль соответствует всем требованиям качества, код сохраняет пользователя с его хешированным паролем в базе данных. Наконец, код возвращает сериализованного сохраненного пользователя в формате JSON в качестве тела с кодом состояния HTTP **201 Created**:

В следующей таблице показан метод наших ранее созданных классов, связанных с **users**, который мы хотим выполнять для каждой комбинации HTTP-команды и области действия.

| HTTP запрос | Область действия        | Класс и метод          | Требует аутентификации |
| ----------- | ----------------------- | ---------------------- | ---------------------- |
| GET         | Коллекция пользователей | UserListResource.get   | Да                     |
| GET         | Пользователь            | UserResource.get       | Да                     |
| POST        | Коллекция пользователей | UserListResource. post | Нет                    |

Мы должны выполнить необходимые настройки маршрутизации ресурсов, чтобы вызвать соответствующие методы и передать им все необходимые аргументы, определив правила URL. Следующие строки настраивают маршрутизацию ресурсов, связанных с пользователем, в **api**. Откройте файл `api/views.py` и добавьте следующие строки в конец кода. Файл кода для примера включен в папку `restful_python_chapter_07_02`:

```python
api.add_resource(UserListResource, '/users/')
api.add_resource(UserResource, '/users/<int:id>')
```

Каждый вызов метода `api.add_resource` направляет URL-адрес к одному из ранее закодированных ресурсов, связанных с пользователем. Когда есть запрос к API и URL-адрес соответствует одному из URL-адресов, указанных в методе `api.add_resource`, Flask вызовет метод, который соответствует запросу HTTP для указанного класса.

## Запуск миграции для создания таблицы пользователей

Теперь мы запустим множество скриптов для выполнения миграций и создания необходимой таблицы в базе данных **PostgreSQL**. Убедитесь, что вы запускаете сценарии в терминале или в окне командной строки, в котором вы активировали виртуальную среду, и что вы находитесь в папке **api**.

Запустите первый сценарий, который заполняет сценарий миграции _**обнаруженными изменениями в моделях**_. В данном случае это второй раз, когда мы заполняем сценарий миграции, поэтому сценарий миграции сгенерирует новую таблицу, которая сохранит нашу новую модель пользователя **User**: **model**:

```bash
python migrate.py db migrate
```

В следующих строках показан образец вывода, сгенерированный после запуска предыдущего скрипта. Ваш вывод будет отличаться в зависимости от базовой папки, в которой вы создали виртуальную среду.

```bash
INFO [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO [alembic.runtime.migration] Will assume transactional DDL.
INFO [alembic.autogenerate.compare] Detected added table 'user'
INFO [alembic.ddl.postgresql] Detected sequence named 'message_id_seq' as
owned by integer column 'message(id)', assuming SERIAL and omitting
Generating
/Users/gaston/PythonREST/Flask02/api/migrations/versions/c8c45e615f6d_.py
... done
```

Вывод показывает, что файл `api/migrations/versions/c8c45e615f6d_.py` содержит код для создания пользовательских таблиц. В следующих строках показан код для этого файла, который был автоматически сгенерирован на основе моделей. Обратите внимание, что имя файла в вашей конфигурации будет другим. Файл кода для примера включен в папку `restful_python_chapter_06_01`:

```
"""empty message
Revision ID: c8c45e615f6d
Revises: 417543056ac3
Create Date: 2016-08-11 17:31:44.989313
"""

# идентификаторы ревизий, используемые Alembic.
revision = 'c8c45e615f6d'
down_revision = '417543056ac3'

from alembic import op
import sqlalchemy as sa

def upgrade():
    ### commands auto generated by Alembic - please adjust! ###
    op.create_table('user',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('name', sa.String(length=50), nullable=False),
        sa.Column('hashed_password', sa.String(length=120), nullable=False),
        sa.Column(
            'creation_date',
            sa.TIMESTAMP(),
            server_default=sa.text('CURRENT_TIMESTAMP'),
            nullable=False),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('name')
    )
    ### конец команд Alembic ###

def downgrade():
    ### команды автоматически генерируются Alembic - пожалуйста, настройте! ###
    op.drop_table('user')
    ### конец команд Alembic ###
```

Код определяет две функции: **upgrade** и **downgrade**. Функция **update** запускает необходимый код для создания пользовательской таблицы **User**, вызывая `alembic.op.create_table`. Функция **downgrade** версии запускает необходимый код, чтобы вернуться к предыдущей версии.

Запустите второй скрипт для обновления базы данных:

```bash
python migrate.py db upgrade
```

В следующих строках показан образец вывода, сгенерированный после запуска предыдущего скрипта:

```bash
INFO [alembic.runtime.migration] Context impl PostgresqlImpl.
INFO [alembic.runtime.migration] Will assume transactional DDL.
INFO [alembic.runtime.migration] Running upgrade 417543056ac3 ->
c8c45e615f6d, empty message
```

Предыдущий скрипт вызывал функцию **upgrade**, определенную в автоматически сгенерированном скрипте `api/migrations/versions/c8c45e615f6d_.py`. Не забывайте, что в вашей конфигурации имя файла будет другим.

После запуска предыдущих сценариев мы можем использовать командную строку PostgreSQL или любое другое приложение, позволяющее нам легко проверить содержимое базы данных PostgreSQL, чтобы проверить новую таблицу, сгенерированную миграцией. Выполните следующую команду, чтобы получить список сгенерированных таблиц. В случае, если используемое вами имя базы данных не является именованным **messages**, убедитесь, что вы используете соответствующее имя базы данных:

```bash
psql --username=user_name --dbname=messages --command="\dt"
```

Следующие строки показывают выходные данные со всеми сгенерированными именами таблиц. Обновление миграции создает новую таблицу с именем **user**.

```bash
             List of relations
 Schema |       Name      |  Type |   Owner
--------+-----------------+-------+-----------
 public | alembic_version | table | user_name
 public | category        | table | user_name
 public | message         | table | user_name
 public | user            | table | user_name
(4 rows)
```

SQLAlchemy сгенерировала пользовательскую таблицу **user** с ее первичным ключом, уникальным ограничением на поле имени и полем пароля на основе информации, включенной в нашу модель пользователя **User**.

Следующая команда позволит вам проверить содержимое таблицы пользователей после того, как мы составим и отправим HTTP-запросы к RESTful API и создадим новых пользователей. Команды предполагают, что вы используете PostgreSQL на том же компьютере, на котором вы выполняете команду:

```bash
psql --username=user_name --dbname=messages --command="SELECT * FROM public.user;"
```

Теперь мы можем запустить скрипт `api/run.py`, который запускает разработку **Flask**. Выполните следующую команду в папке **api**:

```bash
python run.py
```

После того, как мы выполним предыдущую команду, сервер разработки начнет прослушивать порт `5000`.

## Составление запросов с необходимой аутентификацией

Теперь мы составим и отправим HTTP-запрос для получения первой страницы сообщений без учетных данных для аутентификации:

```bash
http POST ':5000/api/messages/?page=1'
```

Ниже приведена эквивалентная команда **curl**:

```bash
curl -iX GET ':5000/api/messages/?page=1'
```

Мы получим код состояния **401 Unauthorized** в заголовке ответа. В следующих строках показан пример ответа:

```http
HTTP/1.0 401 UNAUTHORIZED
Content-Length: 19
Content-Type: text/html; charset=utf-8
Date: Mon, 15 Aug 2016 01:16:36 GMT
Server: Werkzeug/0.11.10 Python/3.5.1
WWW-Authenticate: Basic realm="Authentication Required"
```

Если мы хотим получить сообщения, то есть сделать GET-запрос к `/api/messages/`, нам нужно предоставить учетные данные для аутентификации с использованием HTTP-аутентификации. Однако, прежде чем мы сможем это сделать, необходимо создать нового пользователя. Мы будем использовать нового пользователя для тестирования наших новых классов ресурсов, связанных с пользователями, и наших изменений в политиках разрешений.

```bash
http POST :5000/api/users/ name='brandon' password='brandonpassword'
```

Ниже приведена эквивалентная команда **curl**:

```bash
curl -iX POST -H "Content-Type: application/json" -d '{"name": "brandon",
"password": "brandonpassword"}' :5000/api/users/
```

Конечно, создание пользователя и выполнение методов, требующих аутентификации, должны быть возможны только по протоколу **HTTPS**. Таким образом, имя пользователя и пароль будут зашифрованы.

Предыдущая команда создаст и отправит HTTP-запрос **POST** с указанными парами ключ-значение **JSON**. В запросах указывается `/api/users/`, поэтому он будет соответствовать URL-маршруту `'/users/'` для ресурса **UserList** и запускать метод `UserList.post`, не требующий аутентификации. Метод не получает аргументов, так как URL-маршрут не содержит никаких параметров. Поскольку HTTP-запрос — **POST**, Flask вызывает метод **post**.

Указанный ранее пароль включает только строчные буквы и, следовательно, не удовлетворяет всем качественным требованиям, которые мы указали для паролей в методе `User.check_password_strength_and_hash_if_ok`. Таким образом, мы получим код состояния **400 Bad Request** в заголовке ответа и сообщение об ошибке, указывающее на то, что пароль не соответствует требованию в теле **JSON**. В следующих строках показан пример ответа:

```http
HTTP/1.0 400 BAD REQUEST
Content-Length: 75
Content-Type: application/json
Date: Mon, 15 Aug 2016 01:29:55 GMT
Server: Werkzeug/0.11.10 Python/3.5.1
{
    "error": "The password must include at least one uppercase letter"
}
```

Следующая команда создаст пользователя с действительным паролем:

```bash
http POST :5000/api/users/ name='brandon' password='iA4!V3riS#c^R9'
```

Ниже приведена эквивалентная команда **curl**:

```bash
curl -iX POST -H "Content-Type: application/json" -d '{"name": "brandon",
"password": "iA4!V3riS#c^R9"}' :5000/api/users/
```

Если новый экземпляр пользователя успешно сохраняется в базе данных, вызов вернет код состояния HTTP **201 Created**, а недавно сохраненный пользователь будет сериализован в формате **JSON** в теле ответа. В следующих строках показан пример ответа на HTTP-запросы с новым объектом **User** в ответах **JSON**. Обратите внимание, что ответ включает URL-адрес созданного пользователя и не содержит никакой информации, связанной с паролем.

```http
HTTP/1.0 201 CREATED
Content-Length: 87
Content-Type: application/json
Date: Mon, 15 Aug 2016 01:33:23 GMT
Server: Werkzeug/0.11.10 Python/3.5.1
{
    "id": 1,
    "name": "brandon",
    "url": "http://localhost:5000/api/users/1"
}
```

Мы можем запустить ранее объясненную команду, чтобы проверить содержимое пользовательской таблицы, созданной миграциями в базе данных **PostgreSQL**. Мы заметим, что содержимое поля **hashed\_password** хешируется для новой строки в пользовательской таблице **user**. На следующем снимке экрана показано содержимое новой строки пользовательской таблицы в базе данных PostgreSQL после выполнения HTTP-запроса:

![](../../.gitbook/assets/db\_4.png)

Если мы хотим получить первую страницу сообщений, то есть сделать GET-запрос к `/api/messages/`, нам нужно предоставить учетные данные для аутентификации с использованием HTTP-аутентификации. Теперь мы составим и отправим HTTP-запрос для получения первой страницы сообщений с учетными данными для аутентификации, то есть с именем пользователя и паролем, которые мы недавно создали:

```bash
http -a 'brandon':'iA4!V3riS#c^R9' ':5000/api/messages/?page=1'
```

Ниже приведена эквивалентная команда **curl**:

```bash
curl --user 'brandon':'iA4!V3riS#c^R9' -iX GET ':5000/api/messages/?page=1'
```

Пользователь будет успешно аутентифицирован, и мы сможем обработать запрос на получение первой страницы сообщений. Со всеми изменениями, которые мы внесли в наш API, неаутентифицированные запросы могут создать только нового пользователя.

## Проверьте свои знания

1. Объект `flask.g` - это:
   1. Прокси, предоставляющий доступ к текущему запросу.
   2. Экземпляр класса `flask_httpauth.HTTPBasicAuth`.
   3. Прокси, который позволяет нам хранить на нем все, чем мы хотим поделиться, только для одного запроса.
2. Пакет **passlib** предоставляет:
   1. Фреймворк для хеширования паролей, поддерживающий более 30 схем.
   2. Платформа аутентификации, которая автоматически добавляет модели для пользователей и разрешений в приложение Flask.
   3. Легкий веб-фреймворк, который заменяет Flask.
3. Декоратор `auth.verify_password` применяется к функции:
   1. Делает эту функцию обратным вызовом, который **Flask-HTTPAuth** будет использовать для хеширования пароля для конкретного пользователя.
   2. Делает эту функцию обратным вызовом, который **SQLAlchemy** будет использовать для проверки пароля для определенного пользователя.
   3. Делает эту функцию обратным вызовом, который **Flask-HTTPAuth** будет использовать для проверки пароля для конкретного пользователя.
4. Когда вы назначаете список, который включает `auth.login_required`, свойству **method\_decorators** любого подкласса `flask_restful.Resource`, учитывая, что **auth** является экземпляром `flask_httpauth.HTTPBasicAuth()`:
   1. Ко всем методам, объявленным в ресурсе, будет применен декоратор `auth.login_required`.
   2. К методу **post**, объявленному в ресурсе, будет применен декоратор `auth.login_required`.
   3. К любому из следующих методов, объявленных в ресурсе, будет применен декоратор `auth.login_required`: **delete**, **patch**, **post** и **put**.
5. Какая из следующих строк извлекает целочисленное значение для аргумента `'page'` из объекта запроса, учитывая, что код будет выполняться в методе, определенном в подклассе класса `flask_restful.Resource`?
   1. `page_number = request.get_argument('page', 1, type=int)`
   2. `page_number = request.args.get('page', 1, type=int)`
   3. `page_number = request.arguments.get('page', 1, type=int)`

## Резюме

В этой главе мы улучшили RESTful API во многих отношениях. Мы добавили удобные сообщения об ошибках, когда ресурсы не уникальны. Мы проверили, как обновлять одно или несколько полей с помощью метода **PATCH**, и создали собственный универсальный класс разбивки на страницы.

Затем мы начали работать с аутентификацией и разрешениями. Мы добавили модель пользователя и обновили базу данных. Мы внесли множество изменений в различные фрагменты кода для достижения определенной цели безопасности и воспользовались преимуществами **Flask-HTTPAuth** и **passlib** для использования HTTP-аутентификации в нашем API.

Теперь, когда мы создали улучшенный сложный API, который использует разбивку на страницы и аутентификацию, мы будем использовать дополнительные абстракции, включенные в структуру, и мы будем кодировать, выполнять и улучшать модульные тесты, что мы собираемся обсудить в следующей главе.
