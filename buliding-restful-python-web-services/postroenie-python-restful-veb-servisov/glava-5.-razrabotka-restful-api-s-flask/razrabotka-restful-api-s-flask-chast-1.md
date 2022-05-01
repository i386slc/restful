# Разработка RESTful API с Flask (часть 1)

## Разработка RESTful API для взаимодействия с простым источником данных

Представьте, что нам нужно настроить отображение сообщений на OLED-дисплее, подключенном к устройству IoT (Интернет вещей), устройство IoT способно запускать Python 3.5, Flask и другие пакеты Python. Есть команда, которая пишет код, который извлекает строковые сообщения из словаря и отображает их на OLED-дисплее, подключенном к устройству IoT. Мы должны начать работу над мобильным приложением и веб-сайтом, который должен взаимодействовать с RESTful API для выполнения операций CRUD со строковыми сообщениями.

Нам не нужен ORM, потому что мы не будем сохранять строковые сообщения в базе данных. Мы просто будем работать со словарем в памяти в качестве источника данных. Это одно из требований для этого RESTful API. В этом случае веб-сервис RESTful будет работать на IoT-устройстве, то есть мы будем запускать сервер разработки Flask на IoT-устройстве.

### Совет

Мы определенно потеряем масштабируемость для нашего RESTful API, потому что у нас есть источник данных в памяти на сервере, и поэтому мы не можем запустить RESTful API на другом устройстве IoT. Однако мы будем работать с другим примером, связанным с более сложным источником данных, который позже можно будет масштабировать с помощью RESTful. Первый пример позволит нам понять, как **Flask** и **Flask-RESTful** работают вместе с очень простым источником данных в памяти.

Мы выбрали **Flask**, потому что он легче, чем **Django**, нам не нужно настраивать ORM, и мы хотим как можно скорее запустить RESTful API на устройстве IoT, чтобы все команды могли взаимодействовать с ним. Мы также будем кодировать веб-сайт с помощью Flask, и поэтому мы хотим использовать одну и ту же веб-микроструктуру для обеспечения работы веб-сайта и веб-службы RESTful.

Для **Flask** доступно множество расширений, упрощающих выполнение определенных задач с помощью микрофреймворка Flask. Мы воспользуемся преимуществами **Flask-RESTful**, расширения, которое позволит нам поощрять лучшие практики при создании нашего RESTful API. В этом случае мы будем работать со словарем Python в качестве источника данных. Как объяснялось ранее, в следующих примерах мы будем работать с более сложными источниками данных.

Во-первых, мы должны указать требования к нашему основному ресурсу: сообщению (**message**). Нам нужны следующие атрибуты или поля для сообщения:

* Целочисленный идентификатор
* Строковое сообщение
* Продолжительность в секундах, указывающая время, в течение которого сообщение должно быть напечатано на OLED-дисплее.
* Дата и время создания — временная метка будет добавлена автоматически при добавлении нового сообщения в коллекцию.
* Описание категории сообщения, например «Warning» и «Information».
* Целочисленный счетчик, указывающий, сколько раз сообщение было напечатано на OLED-дисплее.
* Логическое значение, указывающее, было ли сообщение напечатано хотя бы один раз на OLED-дисплее.

В следующей таблице показаны действия HTTP запросов, область действия и семантика методов, которые должна поддерживать наша первая версия API. Каждый метод состоит из запроса HTTP и области действия, и все методы имеют четко определенное значение для всех сообщений и коллекций. В нашем API каждое сообщение имеет свой уникальный URL.

| HTTP запрос | Область действия    | Семантика                                                                                      |
| ----------- | ------------------- | ---------------------------------------------------------------------------------------------- |
| GET         | Коллекция сообщений | Получить все сохраненные сообщения в коллекции, отсортированные по имени в порядке возрастания |
| GET         | Сообщение           | Получить одно сообщение                                                                        |
| POST        | Коллекция сообщений | Создать новое сообщение в коллекции                                                            |
| PATCH       | Сообщение           | Обновить поле для существующего сообщения                                                      |
| DELETE      | Сообщение           | Удалить существующее сообщение                                                                 |

## Понимание задач, выполняемых каждым HTTP-методом

Предположим, что `http://localhost:5000/api/messages/` — это URL для коллекции сообщений. Если мы добавим число к предыдущему URL-адресу, мы идентифицируем конкретное сообщение, идентификатор которого равен указанному числовому значению. Например, `http://localhost:5000/api/messsages/6` идентифицирует сообщение, идентификатор **id** которого равен 6.

{% hint style="info" %}
**СОВЕТ**

Мы хотим, чтобы наш API мог отличать коллекции от одного ресурса этой коллекции в URL-адресах. Когда мы ссылаемся на коллекцию, мы будем использовать косую черту (`/`) в качестве последнего символа URL-адреса, как в `http://localhost:5000/api/messages/`. Когда мы ссылаемся на один ресурс коллекции, мы не будем использовать косую черту (`/`) в качестве последнего символа URL-адреса, как в `http://localhost:5000/api/messages/6`.
{% endhint %}

Мы должны составить и отправить HTTP-запрос **POST** и с URL-адресом запроса `http://localhost:5000/api/messages/`, чтобы создать новое сообщение. Кроме того, мы должны предоставить пары ключ-значение **JSON** с именами полей и значениями для создания нового сообщения. В результате запроса сервер проверит предоставленные значения для полей, удостоверится, что это действительное сообщение, и сохранит его в словаре сообщений.

Сервер вернет код состояния **201 Created** и тело **JSON** с недавно добавленным сообщением, сериализованным в **JSON**, включая назначенный идентификатор **id**, который был автоматически сгенерирован сервером для объекта сообщения:

```http
POST http://localhost:5000/api/messages/
```

Мы должны составить и отправить HTTP-запрос **GET** HTTP и с URL-адресом запроса `http://localhost:5000/api/messages/{id}`, чтобы получить сообщение, идентификатор которого соответствует указанному числовому значению в месте, где `{id}` написано. Например, если мы используем URL-адрес запроса `http://localhost:5000/api/messages/82`, сервер получит игру, **id** которой соответствует 82. В результате запроса сервер получит сообщение с указанным идентификатор из словаря.

Если сообщение найдено, сервер сериализует объект сообщения в **JSON** и возвращает код состояния **200 OK** и тело **JSON** с сериализованным объектом сообщения. Если ни одно сообщение не соответствует указанному идентификатору или первичному ключу, сервер вернет статус **404 Not Found**:

```http
GET http://localhost:5000/api/messages/{id}
```

Мы должны составить и отправить HTTP-запрос **PATCH** и с URL-адресом запроса `http://localhost:5000/api/messages/{id}`, чтобы обновить одно или несколько полей для сообщения, идентификатор которого соответствует указанному числовому значению в место, где написано `{id}`. Кроме того, мы должны предоставить пары ключ-значение **JSON** с именами полей, которые нужно обновить, и их новыми значениями. В результате запроса сервер проверит предоставленные значения для полей, обновит эти поля в сообщении, которое соответствует указанному идентификатору, и обновит сообщение в словаре, если оно является допустимым сообщением.

Сервер вернет код состояния **200 OK** и тело **JSON** с недавно обновленной игрой, сериализованной в **JSON**. Если мы предоставим неверные данные для обновляемых полей, сервер вернет код состояния **400 Bad Request**. Если сервер не найдет сообщение с указанным идентификатором, сервер вернет просто статус **404 Not Found**:

```http
PATCH http://localhost:5000/api/messages/{id}
```

{% hint style="info" %}
**СОВЕТ**

Метод **PATCH** позволит нам легко обновить два поля сообщения: целочисленный счетчик, указывающий, сколько раз сообщение было напечатано, и логическое значение, указывающее, было ли сообщение напечатано хотя бы один раз.
{% endhint %}

Мы должны составить и отправить HTTP-запрос **DELETE** и с URL-адресом запроса `http://localhost:5000/api/messages/{id}`, чтобы удалить сообщение, идентификатор которого соответствует указанному числовому значению в месте, где `{id}` записано. Например, если мы используем URL-адрес запроса `http://localhost:5000/api/messages/15`, сервер удалит сообщение, **id** которого соответствует 15. В результате запроса сервер получит сообщение с указанным идентификатором из словаря. Если сообщение найдено, сервер запросит у словаря удаление записи, связанной с этим объектом сообщения, и вернет код состояния **204 No Content**. Если ни одно сообщение не соответствует указанному идентификатору, сервер вернет статус **404 Not Found**:

```http
DELETE http://localhost:5000/api/messages/{id}
```

## Настройка виртуальной среды с помощью Flask и Flask-RESTful

В [главе 1](../razrabotka-restful-api-s-django.md) «Разработка RESTful API с помощью Django» мы узнали, что в этой книге мы будем работать с облегченными виртуальными средами, представленными и улучшенными в Python 3.4. Теперь мы будем следовать инструкциям по созданию новой облегченной виртуальной среды для работы с Flask и Flask-RESTful. Настоятельно рекомендуется прочитать [главу 1](../razrabotka-restful-api-s-django.md) «Разработка RESTful API с помощью Django», если у вас нет опыта работы с облегченными виртуальными средами на Python. Глава включает в себя все подробные объяснения результатов шагов, которым мы собираемся следовать.

Во-первых, мы должны выбрать целевую папку или каталог для нашей виртуальной среды. Мы будем использовать следующий путь в примере для macOS и Linux. Целевой папкой для виртуальной среды будет папка `PythonREST/Flask01` в нашем домашнем каталоге. Например, если нашим домашним каталогом в macOS или Linux является `/Users/gaston`, виртуальная среда будет создана внутри `/Users/gaston/PythonREST/Flask01`. Вы можете заменить указанный путь желаемым путем в каждой команде, как показано ниже:

```bash
~/PythonREST/Flask01
```

Мы будем использовать следующий путь в примере для Windows. Целевой папкой для виртуальной среды будет папка `PythonREST\Flask01` в папке нашего профиля пользователя. Например, если папка нашего профиля пользователя `C:\Users\Gaston`, виртуальная среда будет создана внутри `C:\Users\gaston\PythonREST\Flask01`. Вы можете заменить указанный путь желаемым путем в каждой команде, как показано ниже:

```powershell
%USERPROFILE%\PythonREST\Flask01
```

Откройте терминал в macOS или Linux и выполните следующую команду, чтобы создать виртуальную среду:

```bash
python3 -m venv ~/PythonREST/Flask01
```

В Windows выполните следующую команду, чтобы создать виртуальную среду:

```powershell
python -m venv %USERPROFILE%\PythonREST\Flask01
```

Предыдущая команда не производит никакого вывода. Теперь, когда мы создали виртуальную среду, мы запустим сценарий для конкретной платформы, чтобы активировать ее. После того, как мы активируем виртуальную среду, мы установим пакеты, которые будут доступны только в этой виртуальной среде.

Если ваш терминал настроен на использование оболочки **bash** в macOS или Linux, выполните следующую команду, чтобы активировать виртуальную среду. Команда также работает для оболочки **zsh**:

```bash
source ~/PythonREST/Flask01/bin/activate
```

Если ваш терминал настроен на использование оболочки **csh** или **tcsh**, выполните следующую команду, чтобы активировать виртуальную среду:

```bash
source ~/PythonREST/Flask01/bin/activate.csh
```

Если ваш терминал настроен на использование оболочки **fish**, выполните следующую команду, чтобы активировать виртуальную среду:

```bash
source ~/PythonREST/Flask01/bin/activate.fish
```

В Windows вы можете запустить пакетный файл в командной строке или сценарий Windows **PowerShell** для активации виртуальной среды. Если вы предпочитаете командную строку, выполните следующую команду в командной строке Windows, чтобы активировать виртуальную среду:

```powershell
%USERPROFILE%\PythonREST\Flask01\Scripts\activate.bat
```

Если вы предпочитаете Windows **PowerShell**, запустите ее и выполните следующие команды, чтобы активировать виртуальную среду. Однако обратите внимание, что вы должны включить выполнение скриптов в Windows **PowerShell**, чтобы иметь возможность запускать скрипт:

```powershell
cd $env:USERPROFILE
PythonREST\Flask01\Scripts\Activate.ps1
```

После активации виртуальной среды в командной строке будет отображаться имя корневой папки виртуальной среды, заключенное в круглые скобки, в качестве префикса для приглашения по умолчанию, чтобы напомнить нам, что мы работаем в виртуальной среде. В этом случае мы увидим `(Flask01)` в качестве префикса для командной строки, потому что корневая папка для активированной виртуальной среды — **Flask01**.

Мы создали и активировали виртуальную среду. Теперь пришло время запустить команды, которые будут одинаковыми для macOS, Linux или Windows; мы должны запустить следующую команду, чтобы установить **Flask-RESTful** с помощью **pip**. **Flask** является зависимостью для **Flask-RESTful**, поэтому **pip** также установит его автоматически:

```bash
pip install flask-restful
```

В последних строках вывода будут указаны все успешно установленные пакеты, включая **flask-restful** и **Flask**:

```bash
Installing collected packages: six, pytz, click, itsdangerous, MarkupSafe,
Jinja2, Werkzeug, Flask, python-dateutil, aniso8601, flask-restful
Running setup.py install for click
Running setup.py install for itsdangerous
Running setup.py install for MarkupSafe
Running setup.py install for aniso8601
Successfully installed Flask-0.11.1 Jinja2-2.8 MarkupSafe-0.23 Werkzeug-
0.11.10 aniso8601-1.1.0 click-6.6 flask-restful-0.3.5 itsdangerous-0.24
python-dateutil-2.5.3 pytz-2016.4 six-1.10.0
```

## Объявление кодов состояния для ответов

Ни **Flask**, ни **Flask-RESTful** не включают объявление переменных для разных кодов состояния HTTP. Мы не хотим возвращать числа как коды состояния. Мы хотим, чтобы наш код было легко читать и понимать, поэтому мы будем использовать описательные коды состояния HTTP. Мы позаимствуем код, который объявляет полезные функции и переменные, связанные с кодами состояния HTTP, из файла `status.py`, включенного в **Django REST Framework**, то есть в структуру, которую мы использовали в предыдущих главах.

Сначала создайте папку с именем **api** в корневой папке недавно созданной виртуальной среды, а затем создайте новый файл **status.py** в папке **api**. В следующих строках показан код, объявляющий функции и переменные с описательными кодами состояния HTTP в файле `api/models.py`, заимствованном из модуля **rest\_framework.status**. Мы не хотим изобретать велосипед, и этот модуль предоставляет все необходимое для работы с кодами состояния HTTP в нашем API на основе **Flask**. Файл кода для примера включен в папку `restful_python_chapter_05_01`:

```python
def is_informational(code):
    return code >= 100 and code <= 199

def is_success(code):
    return code >= 200 and code <= 299

def is_redirect(code):
    return code >= 300 and code <= 399

def is_client_error(code):
    return code >= 400 and code <= 499

def is_server_error(code):
    return code >= 500 and code <= 599

HTTP_100_CONTINUE = 100
HTTP_101_SWITCHING_PROTOCOLS = 101
HTTP_200_OK = 200
HTTP_201_CREATED = 201
HTTP_202_ACCEPTED = 202
HTTP_203_NON_AUTHORITATIVE_INFORMATION = 203
HTTP_204_NO_CONTENT = 204
HTTP_205_RESET_CONTENT = 205
HTTP_206_PARTIAL_CONTENT = 206
HTTP_300_MULTIPLE_CHOICES = 300
HTTP_301_MOVED_PERMANENTLY = 301
HTTP_302_FOUND = 302
HTTP_303_SEE_OTHER = 303
HTTP_304_NOT_MODIFIED = 304
HTTP_305_USE_PROXY = 305
HTTP_306_RESERVED = 306
HTTP_307_TEMPORARY_REDIRECT = 307
HTTP_400_BAD_REQUEST = 400
HTTP_401_UNAUTHORIZED = 401
HTTP_402_PAYMENT_REQUIRED = 402
HTTP_403_FORBIDDEN = 403
HTTP_404_NOT_FOUND = 404
HTTP_405_METHOD_NOT_ALLOWED = 405
HTTP_406_NOT_ACCEPTABLE = 406
HTTP_407_PROXY_AUTHENTICATION_REQUIRED = 407
HTTP_408_REQUEST_TIMEOUT = 408
HTTP_409_CONFLICT = 409
HTTP_410_GONE = 410
HTTP_411_LENGTH_REQUIRED = 411
HTTP_412_PRECONDITION_FAILED = 412
HTTP_413_REQUEST_ENTITY_TOO_LARGE = 413
HTTP_414_REQUEST_URI_TOO_LONG = 414
HTTP_415_UNSUPPORTED_MEDIA_TYPE = 415
HTTP_416_REQUESTED_RANGE_NOT_SATISFIABLE = 416
HTTP_417_EXPECTATION_FAILED = 417
HTTP_428_PRECONDITION_REQUIRED = 428
HTTP_429_TOO_MANY_REQUESTS = 429
HTTP_431_REQUEST_HEADER_FIELDS_TOO_LARGE = 431
HTTP_451_UNAVAILABLE_FOR_LEGAL_REASONS = 451
HTTP_500_INTERNAL_SERVER_ERROR = 500
HTTP_501_NOT_IMPLEMENTED = 501
HTTP_502_BAD_GATEWAY = 502
HTTP_503_SERVICE_UNAVAILABLE = 503
HTTP_504_GATEWAY_TIMEOUT = 504
HTTP_505_HTTP_VERSION_NOT_SUPPORTED = 505
HTTP_511_NETWORK_AUTHENTICATION_REQUIRED = 511
```

Код объявляет пять функций, которые получают код состояния HTTP в аргументе **code** и определяют, к какой из следующих категорий относится код состояния: информационные, успех, перенаправление, ошибки клиента или категории ошибок сервера. Мы будем использовать предыдущие переменные, когда нам нужно будет вернуть определенный код состояния. Например, если нам нужно вернуть код состояния **404 Not Found**, мы вернем `status.HTTP_404_NOT_FOUND` вместо просто **404**.

## Создание модели

Теперь мы создадим простой класс **MessageModel**, который будем использовать для представления сообщений. Помните, что мы не будем сохранять модель в базе данных, и поэтому в этом случае наш класс просто предоставит необходимые атрибуты и не будет отображать информацию. Создайте новый файл **models.py** в папке **api**. В следующих строках показан код, создающий класс **MessageModel** в файле `api/models.py`. Файл кода для примера включен в папку `restful_python_chapter_05_01`:

```
class MessageModel:
    def __init__(self, message, duration, creation_date, message_category):
        # Мы автоматически сгенерируем новый идентификатор id
        self.id = 0
        self.message = message
        self.duration = duration
        self.creation_date = creation_date
        self.message_category = message_category
        self.printed_times = 0
        self.printed_once = False
```

Класс **MessageModel** просто объявляет конструктор, то есть метод **\_\_init\_\_**. Этот метод получает множество аргументов, а затем использует их для инициализации атрибутов с теми же именами: **message**, **duration**, **creation\_date** и **message\_category**. Для атрибута **id** установлено значение `0`, для **print\_times** установлено значение `0`, а для **print\_once** установлено значение `False`. Мы будем автоматически увеличивать идентификатор для каждого нового сообщения, созданного с помощью вызовов API.

## Использование словаря в качестве хранилища

Теперь мы создадим класс **MessageManager**, который будем использовать для сохранения экземпляров **MessageModel** в словаре в памяти. Наши методы API будут вызывать методы класса **MessageManager** для извлечения, вставки, обновления и удаления экземпляров **MessageModel**. Создайте новый файл **api.py** в папке **api**. В следующих строках показан код, создающий класс **MessageManager** в файле `api/api.py`. Кроме того, следующие строки объявляют все импорты, которые нам понадобятся для всего кода, который мы напишем в этом файле. Файл кода для примера включен в папку `restful_python_chapter_05_01`.

```python
from flask import Flask
from flask_restful import abort, Api, fields, marshal_with, reqparse, Resource
from datetime import datetime
from models import MessageModel
import status
from pytz import utc

class MessageManager():
    last_id = 0
    def __init__(self):
        self.messages = {}

    def insert_message(self, message):
        self.__class__.last_id += 1
        message.id = self.__class__.last_id
        self.messages[self.__class__.last_id] = message

    def get_message(self, id):
        return self.messages[id]

    def delete_message(self, id):
        del self.messages[id]
```

Класс **MessageManager** объявляет атрибут класса **last\_id** и инициализирует его значением `0`. Этот атрибут класса хранит последний идентификатор, который был сгенерирован и присвоен экземпляру **MessageModel**, хранящемуся в словаре. Конструктор, то есть метод **\_\_init\_\_**, создает и инициализирует атрибут **messages** как пустой словарь.

Код объявляет следующие три метода для класса:

* **insert\_message**: этот метод получает недавно созданный экземпляр **MessageModel** в аргументе сообщения. Код увеличивает значение атрибута класса **last\_id**, а затем присваивает полученное значение идентификатору полученного сообщения. Код использует **self.\_\_class\_\_** для ссылки на тип текущего экземпляра. Наконец, код добавляет сообщение в качестве значения к ключу, идентифицированному сгенерированным идентификатором, **last\_id**, в словаре **self.messages**.
* **get\_message**: этот метод получает идентификатор сообщения, которое необходимо получить из словаря **self.messages**. Код возвращает значение, связанное с ключом, который соответствует полученному идентификатору в словаре **self.messages**, который мы используем в качестве источника данных.
* **delete\_message**: этот метод получает идентификатор сообщения, которое необходимо удалить из словаря **self.messages**. Код удаляет пару ключ-значение, ключ которой соответствует полученному идентификатору в словаре **self.messages**, который мы используем в качестве источника данных.

Нам не нужен метод для обновления сообщения, потому что мы просто внесем изменения в атрибуты экземпляра **MessageModel**, который уже хранится в словаре **self.messages**. Значение, хранящееся в словаре, является ссылкой на экземпляр **MessageModel**, который мы обновляем, поэтому нам не нужно вызывать определенный метод для обновления экземпляра в словаре. Однако, если бы мы работали с базой данных, нам нужно было бы вызвать метод обновления для нашего ORM или хранилища данных.

## Настройка полей вывода

Теперь мы создадим словарь **message\_fields**, который будем использовать для управления данными, которые мы хотим, чтобы **Flask-RESTful** отображал в нашем ответе, когда мы возвращаем экземпляры **MessageModel**. Откройте ранее созданный файл `api/api.py` и добавьте следующие строки. Файл кода для примера включен в папку `restful_python_chapter_05_01`.

```python
message_fields = {
    'id': fields.Integer,
    'uri': fields.Url('message_endpoint'),
    'message': fields.String,
    'duration': fields.Integer,
    'creation_date': fields.DateTime,
    'message_category': fields.String,
    'printed_times': fields.Integer,
    'printed_once': fields.Boolean
}

message_manager = MessageManager()
```

Мы объявили словарь **message\_fields** (dict) с парами строк ключ-значение и классами, объявленными в модуле **flask\_restful.fields**. Ключи — это имена атрибутов, которые мы хотим отобразить из класса **MessageModel**, а значения — это классы, которые форматируют и возвращают значение для поля. В предыдущем коде мы работали со следующими классами, которые форматируют и возвращают значение для указанного поля в ключе:

* **fields.Integer**: выводит целочисленное значение.
* **fields.Url**: генерирует строковое представление URL. По умолчанию этот класс создает относительный URI для запрашиваемого ресурса. Код указывает '**message\_endpoint**' в качестве аргумента конечной точки. Таким образом, класс будет использовать указанное имя конечной точки. Мы объявим эту конечную точку позже в файле **api.py**. Мы не хотим включать имя хоста в сгенерированный URI, поэтому мы используем значение по умолчанию для атрибута absolute bool, которое равно `False`.
* **fields.DateTime**: выводит отформатированную строку даты и времени в формате UTC в формате RFC 822 по умолчанию.
* **fields.Boolean**: генерирует строковое представление логического значения.

Поле '**uri**' использует **fields.Url** и связано с указанной конечной точкой, а не с атрибутом класса **MessageModel**. Это единственный случай, когда указанное имя поля не имеет атрибута в классе **MessageModel**. Другие строки, указанные как ключи, указывают все атрибуты, которые мы хотим отображать в выводе, когда мы используем словарь **message\_fields** для создания окончательного сериализованного вывода ответа.

После объявления словаря **message\_fields** следующая строка кода создает экземпляр ранее созданного класса **MessageManager** с именем **message\_manager**. Мы будем использовать этот экземпляр для создания, извлечения и удаления экземпляров **MessageModel**.

## Работа с ресурсной маршрутизацией поверх подключаемых представлений Flask

**Flask-RESTful** использует ресурсы, созданные поверх подключаемых представлений **Flask**, в качестве основного строительного блока для RESTful API. Нам просто нужно создать подкласс класса `flask_restful.Resource` и объявить методы для каждого поддерживаемого HTTP запроса. Подкласс `flask_restful.Resource` представляет ресурс **RESTful**, поэтому нам придется объявить один класс для представления набора сообщений, а другой — для представления ресурса сообщений.

Во-первых, мы создадим класс **Message**, который будем использовать для представления ресурса сообщения. Откройте ранее созданный файл `api/api.py` и добавьте следующие строки. Файл кода для примера включен в папку `restful_python_chapter_05_01`, как показано ниже:

```python
class Message(Resource):
    def abort_if_message_doesnt_exist(self, id):
        if id not in message_manager.messages:
            abort(
                status.HTTP_404_NOT_FOUND,
                message="Message {0} doesn't exist".format(id))

    @marshal_with(message_fields)
    def get(self, id):
        self.abort_if_message_doesnt_exist(id)
        return message_manager.get_message(id)

    def delete(self, id):
        self.abort_if_message_doesnt_exist(id)
        message_manager.delete_message(id)
        return '', status.HTTP_204_NO_CONTENT

    @marshal_with(message_fields)
    def patch(self, id):
        self.abort_if_message_doesnt_exist(id)
        message = message_manager.get_message(id)
        parser = reqparse.RequestParser()
        parser.add_argument('message', type=str)
        parser.add_argument('duration', type=int)
        parser.add_argument('printed_times', type=int)
        parser.add_argument('printed_once', type=bool)
        args = parser.parse_args()
        if 'message' in args:
            message.message = args['message']
        if 'duration' in args:
            message.duration = args['duration']
        if 'printed_times' in args:
            message.printed_times = args['printed_times']
        if 'printed_once' in args:
            message.printed_once = args['printed_once']
        return message
```

Класс **Message** является подклассом `flask_restful.Resource` и объявляет следующие три метода, которые будут вызываться при поступлении HTTP-метода с тем же именем в качестве запроса на представленный ресурс:

* **get**: этот метод получает идентификатор сообщения **id**, которое необходимо получить в аргументе идентификатора **id**. Код вызывает метод `self.abort_if_message_doesnt_exist` для прерывания в случае отсутствия сообщения с запрошенным идентификатором. Если сообщение существует, код возвращает экземпляр **MessageModel**, идентификатор которого соответствует указанному идентификатору, возвращенному методом `message_manager.get_message`. Метод **get** использует декоратор `@marshal_with` с **message\_fields** в качестве аргумента. Декоратор возьмет экземпляр **MessageModel** и применит фильтрацию полей и форматирование вывода, указанные в **message\_fields**.
* **delete**: этот метод получает идентификатор сообщения **id**, которое необходимо удалить, в аргументе идентификатора. Код вызывает метод `self.abort_if_message_doesnt_exist` для прерывания в случае отсутствия сообщения с запрошенным идентификатором. В случае, если сообщение существует, код вызывает метод `message_manager.delete_message` с полученным идентификатором в качестве аргумента для удаления экземпляра **MessageModel** из нашего хранилища данных. Затем код возвращает пустое тело ответа и код состояния **204 No Content**.
* **patch**: этот метод получает идентификатор сообщения, которое должно быть обновлено или исправлено, в аргументе **id**. Код вызывает метод `self.abort_if_message_doesnt_exist` для прерывания в случае отсутствия сообщения с запрошенным идентификатором. Если сообщение существует, код сохраняет экземпляр **MessageModel**, идентификатор которого соответствует указанному идентификатору, возвращенному методом `message_manager.get_message`, в переменной сообщения. Следующая строка создает экземпляр `flask_restful.reqparse.RequestParser` с именем **parser**. Экземпляр **RequestParser** позволяет нам добавлять аргументы с их именами и типами, а затем легко анализировать аргументы, полученные с запросом. Код делает четыре вызова `parser.add_argument` с именем аргумента и типом четырех аргументов, которые мы хотим проанализировать. Затем код вызывает метод `parser.parse_args` для анализа всех аргументов запроса и сохраняет возвращенный словарь (dict) в переменной **args**. Код обновляет все атрибуты, имеющие новые значения в словаре **args** в экземпляре **MessageModel**: **message**. В случае, если запрос не включал значения для определенных полей, код не будет вносить изменения в реальные атрибуты. Запрос не требует включения четырех полей, которые могут быть обновлены значениями. Код возвращает обновленное сообщение. Метод **patch** использует декоратор `@marshal_with` с **message\_fields** в качестве аргумента. Декоратор возьмет экземпляр **MessageModel**, сообщение и применит фильтрацию полей и форматирование вывода, указанные в **message\_fields**.

{% hint style="info" %}
**СОВЕТ**

Мы использовали несколько возвращаемых значений для установки кода ответа.
{% endhint %}

Как объяснялось ранее, эти три метода вызывают внутренний метод **abort\_if\_message\_doesnt\_exist**, который получает идентификатор существующего экземпляра **MessageModel** в аргументе идентификатора **id**. Если полученный идентификатор отсутствует в ключах словаря `message_manager.messages`, метод вызывает функцию `flask_restful.abort` со статусом `status.HTTP_404_NOT_FOUND` в качестве аргумента **http\_status\_code** и сообщением о том, что сообщение с указанным идентификатором не существует. Функция прерывания вызывает **HTTPException** для полученного **http\_status\_code** и присоединяет к исключению дополнительные аргументы ключевого слова для последующей обработки. В этом случае мы генерируем код состояния **HTTP 404 Not Found**.

Оба метода **get** и **patch** используют декоратор `@marshal_with`, который принимает один объект данных или список объектов данных и применяет фильтрацию поля и форматирование вывода, указанные в качестве аргумента. Маршаллинг также может работать со словарями (dicts). В обоих методах мы указали **message\_fields** в качестве аргумента, и поэтому код отображает следующие поля: **id**, **uri**, **message**, **duration**, **create\_date**, **message\_category**, **print\_times** и **print\_once**. Когда мы используем декоратор `@marshal_with`, мы автоматически возвращаем код состояния **HTTP 200 OK**.

Следующий оператор **return** с декоратором `@marshal_with(message_fields)` возвращает код состояния **HTTP 200 OK**, поскольку мы не указали код состояния после возвращаемого объекта (**message**):

```python
return message
```

Следующая строка — это строка кода, которая действительно выполняется с помощью декоратора `@marshal_with(message_fields)`, и мы можем использовать ее вместо работы с декоратором:

```python
return marshal(message, resource_fields), status.HTTP_200_OK
```

Например, мы можем вызвать функцию **marshal**, как показано в предыдущей строке, вместо декоратора `@marshal_with`, и код даст тот же результат.

Теперь мы создадим класс **MessageList**, который будем использовать для представления коллекции сообщений. Откройте ранее созданный файл `api/api.py` и добавьте следующие строки. Файл кода для примера включен в папку `restful_python_chapter_05_01`:

```python
class MessageList(Resource):
    @marshal_with(message_fields)
    def get(self):
        return [v for v in message_manager.messages.values()]

    @marshal_with(message_fields)
    def post(self):
        parser = reqparse.RequestParser()
        parser.add_argument(
            'message', type=str, required=True,
            help='Message cannot be blank!')
        parser.add_argument(
            'duration', type=int, required=True,
            help='Duration cannot be blank!')
        parser.add_argument(
            'message_category', type=str, required=True,
            help='Message category cannot be blank!')
        args = parser.parse_args()
        message = MessageModel(
            message=args['message'],
            duration=args['duration'],
            creation_date=datetime.now(utc),
            message_category=args['message_category']
        )
        message_manager.insert_message(message)
        return message, status.HTTP_201_CREATED
```

Класс **MessageList** является подклассом `flask_restful.Resource` и объявляет следующие два метода, которые будут вызываться при поступлении HTTP-метода с тем же именем в качестве запроса на представленный ресурс:

* **get**: этот метод возвращает список со всеми экземплярами **MessageModel**, сохраненными в словаре `message_manager.messages`. Метод **get** использует декоратор `@marshal_with` с **message\_fields** в качестве аргумента. Декоратор возьмет каждый экземпляр **MessageModel** в возвращаемом списке и применит фильтрацию полей и форматирование вывода, указанные в **message\_fields**.
* **post**: этот метод создает экземпляр `flask_restful.reqparse.RequestParser` с именем **parser**. Экземпляр **RequestParser** позволяет нам добавлять аргументы с их именами и типами, а затем легко анализировать аргументы, полученные с запросом **POST**, для создания нового экземпляра **MessageModel**. Код делает три вызова `parser.add_argument` с именем аргумента и типом трех аргументов, которые мы хотим проанализировать. Затем код вызывает метод `parser.parse_args` для анализа всех аргументов запроса и сохраняет возвращенный словарь (dict) в переменной **args**. Код использует проанализированные аргументы в словаре, чтобы указать значения для атрибутов **message**, **duration** и **message\_category**, чтобы создать новый экземпляр **MessageModel** и сохранить его в переменной **message**. В качестве значения аргумента **create\_date** устанавливается текущая дата и время с информацией о часовом поясе, поэтому он не анализируется из запроса. Затем код вызывает метод `message_manager.insert_message` с новым экземпляром **MessageModel** (**message**), чтобы добавить этот новый экземпляр в словарь. Метод **post** использует `@marshal_with` декоратор с **message\_fields** в качестве аргумента. Декоратор возьмет недавно созданный и сохраненный экземпляр **MessageModel**, сообщение **message** и применит фильтрацию полей и форматирование вывода, указанные в **message\_fields**. Код возвращает код состояния **HTTP 201 Created**.

В следующей таблице показаны методы наших ранее созданных классов, который мы хотим выполнить для каждой комбинации HTTP запроса и области видимости:

| HTTP запрос | Область видимости   | Класс и метод    |
| ----------- | ------------------- | ---------------- |
| GET         | Коллекция сообщений | MessageList.get  |
| GET         | Сообщение           | Message.get      |
| POST        | Коллекция сообщений | MessageList.post |
| PATCH       | Сообщение           | Message.patch    |
| DELETE      | Сообщение           | Message.delete   |

Если запрос приводит к вызову ресурса с неподдерживаемым методом HTTP, **Flask-RESTful** вернет ответ с кодом состояния HTTP **405 Method Not Allowed**.

## Настройка маршрутизации ресурсов и конечных точек (endpoints)

Мы должны выполнить необходимые настройки маршрутизации ресурсов, чтобы вызвать соответствующие методы и передать им все необходимые аргументы, определив правила URL. Следующие строки создают основную точку входа для приложения, инициализируют его с помощью приложения **Flask** и настраивают маршрутизацию ресурсов для API. Откройте ранее созданный файл `api/api.py` и добавьте следующие строки. Файл кода для примера включен в папку `restful_python_chapter_05_01`:

```python
app = Flask(__name__)
api = Api(app)
api.add_resource(MessageList, '/api/messages/')
api.add_resource(Message, '/api/messages/<int:id>', endpoint='message_endpoint')

if __name__ == '__main__':
    app.run(debug=True)
```

Код создает экземпляр класса `flask_restful.Api` и сохраняет его в переменной **api**. Каждый вызов метода `api.add_resource` направляет URL-адрес к ресурсу, в частности к одному из ранее объявленных подклассов класса `flask_restful.Resource`. Когда есть запрос к API и URL-адрес соответствует одному из URL-адресов, указанных в методе `api.add_resource`, **Flask** вызовет метод, который соответствует запросу HTTP в запросе для указанного класса. Метод следует стандартным правилам маршрутизации **Flask**.

Например, следующая строка сделает HTTP-запрос **GET** к `/api/messages/` без каких-либо дополнительных параметров для вызова метода `MessageList.get`:

```python
api.add_resource(MessageList, '/api/messages/')
```

**Flask** передаст переменные URL вызываемому методу в качестве аргументов. Например, следующая строка отправит HTTP-запрос **GET** к `/api/messages/12` для вызова метода `Message.get` с **12**, переданным в качестве значения для аргумента **id**:

```python
api.add_resource(Message, '/api/messages/<int:id>', endpoint='message_endpoint')
```

Кроме того, мы можем указать строковое значение для аргумента конечной точки, чтобы упростить ссылку на указанный маршрут в полях `fields.Url`. Мы передаем одно и то же имя конечной точки «**message\_endpoint**» в качестве аргумента в поле **uri**, объявленном как `fields.Url` в словаре **message\_fields**, который мы используем для визуализации каждого экземпляра **MessageModel**. Таким образом, `fields.Url` сгенерирует URI с учетом этого маршрута.

Нам потребовалось всего несколько строк кода для настройки маршрутизации ресурсов и конечных точек. Последняя строка просто вызывает метод `app.run` для запуска приложения **Flask** с аргументом отладки, установленным в `True`, чтобы включить отладку. В этом случае мы запускаем приложение, вызывая метод **run** для немедленного запуска локального сервера. Мы также могли бы достичь той же цели, используя сценарий командной строки **flask**. Однако этот вариант потребует от нас настройки переменных среды, а инструкции для платформ, которые мы рассматриваем в этой книге, различаются: macOS, Windows и Linux.

{% hint style="info" %}
**СОВЕТ**

Как и в случае с любой другой веб-платформой, вы никогда не должны включать отладку в производственной среде.
{% endhint %}