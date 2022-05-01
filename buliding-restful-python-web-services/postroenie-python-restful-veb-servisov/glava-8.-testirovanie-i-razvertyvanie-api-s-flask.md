# Глава 8. Тестирование и развертывание API с Flask

В этой главе мы настроим, напишем и выполним модульные тесты и узнаем несколько вещей, связанных с развертыванием. Мы будем:

* Настраивать модульные тесты
* Создавать базу данных для тестирования
* Писать первую часть модульных тестов
* Запускать модульные тесты и проверять охват тестирования
* Улучшать охват тестирования
* Использовать понятия стратегий развертывания и масштабируемости

## Настройка модульных тестов (unit tests)

Мы будем использовать **nose2**, чтобы упростить обнаружение и запуск модульных тестов. Мы будем измерять тестовое покрытие, и поэтому мы установим необходимый пакет, который позволит нам запустить покрытие с помощью **nose2**. Сначала мы установим пакеты **nose2** и **cov-core** в нашу виртуальную среду. Пакет **cov-core** позволит нам измерить тестовое покрытие с помощью **nose2**. Затем мы создадим новую базу данных **PostgreSQL**, которую будем использовать для тестирования. Наконец, мы создадим файл конфигурации для среды тестирования.

Убедитесь, что вы вышли из сервера разработки **Flask**. Помните, что вам просто нужно нажать **Ctrl + C** в терминале или окне командной строки, в котором он запущен. Нам просто нужно запустить следующую команду для установки пакета **nose2**:

```bash
pip install nose2
```

Последние строки вывода будут означать, что пакет **nose2** был успешно установлен.

```bash
Collecting nose2
Collecting six>=1.1 (from nose2)
Downloading six-1.10.0-py2.py3-none-any.whl
Installing collected packages: six, nose2
Successfully installed nose2-0.6.5 six-1.10.0
```

Нам просто нужно запустить следующую команду, чтобы установить пакет **cov-core**, который также установит зависимость **coverage**:

```bash
pip install cov-core
```

Последние строки вывода будут означать, что пакет **cov-core** был успешно установлен:

```bash
Collecting cov-core
Collecting coverage>=3.6 (from cov-core)
Installing collected packages: coverage, cov-core
Successfully installed cov-core-1.15.0 coverage-4.2
```

Теперь мы создадим базу данных **PostgreSQL**, которую будем использовать в качестве репозитория для нашей тестовой среды. Вам нужно будет загрузить и установить базу данных **PostgreSQL**, если вы еще не запустили ее в тестовой среде на своем компьютере или на тестовом сервере.

{% hint style="info" %}
**СОВЕТ**

Не забудьте убедиться, что папка **bin** PostgreSQL включена в переменную окружения **PATH**. Вы должны иметь возможность запускать утилиту командной строки **psql** из текущего терминала или командной строки.
{% endhint %}

Мы будем использовать инструменты командной строки PostgreSQL для создания новой базы данных с именем **test\_messages**. Если у вас уже есть база данных PostgreSQL с таким именем, убедитесь, что вы используете другое имя во всех командах и конфигурациях. Вы можете выполнить ту же задачу с помощью любого графического инструмента PostgreSQL. Если вы разрабатываете в Linux, необходимо запускать команды от имени пользователя **postgres**. Выполните следующую команду в macOS или Windows, чтобы создать новую базу данных с именем **test\_messages**. Обратите внимание, что команда не будет генерировать никакого вывода:

```bash
createdb test_messages
```

В Linux выполните следующую команду, чтобы использовать пользователя **postgres**:

```bash
sudo -u postgres createdb test_messages
```

Теперь мы будем использовать инструмент командной строки **psql** для запуска некоторых операторов SQL, чтобы предоставить пользователю привилегии в базе данных. Если вы используете сервер, отличный от сервера разработки, вам нужно будет создать пользователя перед предоставлением привилегий. В macOS или Windows выполните следующую команду, чтобы запустить **psql**:

```powershell
psql
```

В Linux выполните следующую команду, чтобы использовать пользователя **postgres**

```bash
sudo -u psql
```

Затем выполните следующие операторы SQL и, наконец, введите `\q`, чтобы выйти из инструмента командной строки **psql**. Замените **user\_name** на желаемое имя пользователя для использования в новой базе данных и **password** на выбранный вами пароль. Мы будем использовать имя пользователя и пароль в тестовой конфигурации **Flask**. Вам не нужно выполнять шаги, если вы уже работаете с конкретным пользователем в PostgreSQL и уже предоставили права доступа к базе данных для пользователя:

```sql
GRANT ALL PRIVILEGES ON DATABASE test_messages TO user_name;
\q
```

Создайте новый файл **test\_config.py** в папке **api**. В следующих строках показан код, объявляющий переменные, определяющие конфигурацию **Flask** и **SQLAlchemy** для нашей тестовой среды. Переменная **SQL\_ALCHEMY\_DATABASE\_URI** генерирует URI SQLAlchemy для базы данных **PostgreSQL**, который мы будем использовать для выполнения всех миграций перед запуском тестов, и мы будем удалять все элементы после выполнения всех тестов. Убедитесь, что вы указали желаемое имя тестовой базы данных в значении для **DB\_NAME** и что вы настроили пользователя, пароль, хост и порт в соответствии с вашей конфигурацией PostgreSQL для тестовой среды. Если вы выполнили предыдущие шаги, используйте настройки, указанные в этих шагах. Файл кода для примера включен в папку `restful_python_chapter_08_01`.

```python
import os

basedir = os.path.abspath(os.path.dirname(__file__))
DEBUG = True
PORT = 5000
HOST = "127.0.0.1"
SQLALCHEMY_ECHO = False
SQLALCHEMY_TRACK_MODIFICATIONS = True
SQLALCHEMY_DATABASE_URI = "postgresql://{DB_USER}:{DB_PASS}@{DB_ADDR}/{DB_NAME}".format(
    DB_USER="user_name", DB_PASS="password",
    DB_ADDR="127.0.0.1", DB_NAME="test_messages")
SQLALCHEMY_MIGRATE_REPO = os.path.join(basedir, 'db_repository')
TESTING = True
SERVER_NAME = '127.0.0.1:5000'
PAGINATION_PAGE_SIZE = 5
PAGINATION_PAGE_ARGUMENT_NAME = 'page'
# Отключите защиту CSRF в тестовой конфигурации.
WTF_CSRF_ENABLED = False
```

Как и в случае с аналогичным тестовым файлом, который мы создали для нашей среды разработки, мы укажем ранее созданный модуль в качестве аргумента функции, которая создаст приложение **Flask**, которое мы будем использовать для тестирования. Таким образом, у нас есть один модуль, который указывает все значения для различных переменных конфигурации для нашей среды тестирования, и другой модуль, который создает приложение Flask для нашей среды тестирования. Также возможно создать иерархию классов с одним классом для каждой среды, которую мы хотим использовать. Однако в нашем примере проще создать новый файл конфигурации для нашей тестовой среды.

## Написание первого этапа модульных тестов

Теперь мы напишем первый этап модульных тестов. В частности, мы напишем модульные тесты, связанные с ресурсами пользователей и категорий сообщений: **UserResource**, **UserListResource**, **CategoryResource** и **CategoryListResource**. Создайте новую подпапку с тестами в папке **api**. Затем создайте новый файл **test\_views.py** в новой подпапке `api/tests`. Добавьте следующие строки, которые объявляют множество операторов импорта и первые методы для класса **InitialTests**. Файл кода для примера включен в папку `restful_python_chapter_08_01`:

```python
from app import create_app
from base64 import b64encode
from flask import current_app, json, url_for
from models import db, Category, Message, User
import status
from unittest import TestCase

class InitialTests(TestCase):
    def setUp(self):
        self.app = create_app('test_config')
        self.test_client = self.app.test_client()
        self.app_context = self.app.app_context()
        self.app_context.push()
        self.test_user_name = 'testuser'
        self.test_user_password = 'T3s!p4s5w0RDd12#'
        db.create_all()

    def tearDown(self):
        db.session.remove()
        db.drop_all()
        self.app_context.pop()

    def get_accept_content_type_headers(self):
        return {
            'Accept': 'application/json',
            'Content-Type': 'application/json'
        }

    def get_authentication_headers(self, username, password):
        authentication_headers = self.get_accept_content_type_headers()
        authentication_headers['Authorization'] = \
            'Basic ' + b64encode((username + ':' + password).encode('utf-8')).decode(
                'utf-8'
            )
        return authentication_headers
```

Класс **InitialTests** является подклассом `unittest.TestCase`. Класс переопределяет метод **setUp**, который будет выполняться перед запуском каждого тестового метода. Метод вызывает функцию **create\_app**, объявленную в модуле приложения, с `'test_config'` в качестве аргумента. Функция настроит приложение Flask с этим модулем в качестве файла конфигурации, и поэтому приложение будет использовать ранее созданный файл конфигурации, в котором указаны желаемые значения для нашей тестовой базы данных и среды. Затем код устанавливает для атрибута тестирования недавно созданного приложения значение `True`, чтобы исключение распространилось на тестовый клиент.

Следующая строка вызывает метод `self.app.test_client` для создания тестового клиента для ранее созданного приложения Flask и сохраняет тестовый клиент в атрибуте **test\_client**. Мы будем использовать тестовый клиент в наших методах тестирования, чтобы легко составлять и отправлять запросы к нашему API. Затем код сохраняет и отправляет контекст приложения и создает два атрибута с именем пользователя и паролем, которые мы будем использовать для наших тестов. Наконец, метод вызывает метод `db.create_all` для создания всех необходимых таблиц в нашей тестовой базе данных, настроенной в файле **test\_config.py**.

Класс **InitialTests** переопределяет метод **tearDown**, который будет выполняться после запуска каждого тестового метода. Код удаляет сеанс **SQLAlchemy**, удаляет все таблицы, которые мы создали в тестовой базе данных перед началом выполнения тестов, и извлекает контекст приложения. Таким образом, после завершения выполнения каждого теста тестовая база данных снова будет пустой.

Метод **get\_accept\_content\_type\_headers** создает и возвращает словарь (**dict**) со значениями ключей заголовков **Accept** и **Content-Type**, для которых установлено значение `«application/json»`. Мы будем вызывать этот метод в наших тестах всякий раз, когда нам нужно создать заголовок для составления наших запросов без аутентификации.

Метод **get\_authentication\_headers** вызывает описанный ранее метод **get\_accept\_content\_type\_headers** для создания пар ключ-значение заголовка без аутентификации. Затем код добавляет необходимое значение к ключу авторизации с соответствующей кодировкой, чтобы предоставить имя пользователя и пароль, полученные в аргументах **username** и **password**. Последняя строка возвращает сгенерированный словарь, который включает информацию об аутентификации. Мы будем вызывать этот метод в наших тестах всякий раз, когда нам нужно создать заголовок для составления наших запросов с аутентификацией. Мы будем использовать имя пользователя и пароль, которые мы сохранили в атрибутах метода **setUp**.

Откройте ранее созданный файл **test\_views.py** в новой подпапке `api/tests`. Добавьте следующие строки, которые объявляют множество методов для класса **InitialTests**. Файл кода для примера включен в папку `restful_python_chapter_08_01`.

```python
def test_request_without_authentication(self):
    """
    Убедитесь, что мы не можем получить доступ к ресурсу,
    требующему аутентификации, без соответствующего заголовка аутентификации.
    """
    response = self.test_client.get(
        url_for('api.messagelistresource', _external=True),
        headers=self.get_accept_content_type_headers())
    self.assertTrue(response.status_code == status.HTTP_401_UNAUTHORIZED)

def create_user(self, name, password):
    url = url_for('api.userlistresource', _external=True)
    data = {'name': name, 'password': password}
    response = self.test_client.post(
        url,
        headers=self.get_accept_content_type_headers(),
        data=json.dumps(data))
    return response

def create_category(self, name):
    url = url_for('api.categorylistresource', _external=True)
    data = {'name': name}
    response = self.test_client.post(
        url,
        headers=self.get_authentication_headers(self.test_user_name,
            self.test_user_password),
    data=json.dumps(data))
    return response

def test_create_and_retrieve_category(self):
    """
    Убедитесь, что мы можем создать новую категорию Category, а затем получить ее
    """
    create_user_response = self.create_user(self.test_user_name,
        self.test_user_password)
    self.assertEqual(create_user_response.status_code, status.HTTP_201_CREATED)
    new_category_name = 'New Information'
    post_response = self.create_category(new_category_name)
    self.assertEqual(post_response.status_code, status.HTTP_201_CREATED)
    self.assertEqual(Category.query.count(), 1)
    post_response_data = json.loads(post_response.get_data(as_text=True))
    self.assertEqual(post_response_data['name'], new_category_name)
    new_category_url = post_response_data['url']
    get_response = self.test_client.get(
        new_category_url,
        headers=self.get_authentication_headers(self.test_user_name,
        self.test_user_password))
    get_response_data = json.loads(get_response.get_data(as_text=True))
    self.assertEqual(get_response.status_code, status.HTTP_200_OK)
    self.assertEqual(get_response_data['name'], new_category_name)
```

Метод **test\_request\_without\_authentication** проверяет, был ли нам отказано в доступе к ресурсу, который требует аутентификации, когда мы не предоставляем соответствующий заголовок аутентификации с запросом. Метод использует тестовый клиент для составления и отправки HTTP-запроса **GET** на URL-адрес, сгенерированный для ресурса `«api.messagelistresource»`, чтобы получить список сообщений. Нам нужен аутентифицированный запрос для получения списка сообщений. Однако код вызывает метод **get\_authentication\_headers**, чтобы установить значение аргумента заголовков в вызове `self.test_client.get`, и поэтому код генерирует запрос без аутентификации. Наконец, метод использует **assertTrue** для проверки того, что **status\_code** для ответа имеет значение HTTP **401 Unauthorized** (`status.HTTP_401_UNAUTHORIZED`).

Метод **create\_user** использует тестовый клиент для составления и отправки HTTP-запроса **POST** на URL-адрес, сгенерированный для ресурса `«api.userlistresource»`, чтобы создать нового пользователя с именем и паролем, полученными в качестве аргументов. Нам не нужен аутентифицированный запрос для создания нового пользователя, и поэтому код вызывает ранее объясненный **get\_accept\_content\_type\_headers** для установки значения аргумента заголовков в вызове `self.test_client.post`. Наконец, код возвращает ответ на запрос **POST**. Всякий раз, когда нам нужно создать аутентифицированный запрос, мы будем вызывать метод **create\_user** для создания нового пользователя.

Метод **create\_category** использует тестовый клиент для составления и отправки HTTP-запроса **POST** на URL-адрес, сгенерированный для ресурса `«api.categorylistresource»`, для создания новой категории с именем, полученным в качестве аргумента. Нам нужен аутентифицированный запрос для создания новой категории, и поэтому код вызывает ранее объясненный метод **get\_authentication\_headers**, чтобы установить значение для аргумента заголовков в вызове `self.test_client.post`. Имя пользователя и пароль установлены как `self.test_user_name` и `self.test_user_password`. Наконец, код возвращает ответ на запрос **POST**. Всякий раз, когда нам нужно создать категорию, мы будем вызывать метод **create\_category** после создания соответствующего пользователя, который аутентифицирует запрос.

Метод **test\_create\_and\_retrieve\_category** проверяет, можем ли мы создать новую категорию, а затем получить ее. Этот метод вызывает описанный ранее метод **create\_user** для создания нового пользователя, а затем использует его для аутентификации HTTP-запроса **POST**, сгенерированного в методе **create\_game\_category**. Затем код составляет и отправляет метод HTTP **GET** для извлечения недавно созданной категории с URL-адресом, полученным в ответ на предыдущий запрос HTTP **POST**. Метод использует **assertEqual** для проверки следующих ожидаемых результатов:

* Код состояния для ответа HTTP **POST** — HTTP **201 Created** (`status.HTTP_201_CREATED`).
* Общее количество объектов **Category**, извлеченных из базы данных, равно 1.
* Код состояния для ответа HTTP **GET** — HTTP **200 OK** (`status.HTTP_200_OK`).
* Значение ключа имени в ответе HTTP **GET** равно имени, указанному для новой категории.

Откройте ранее созданный файл **test\_views.py** в новой подпапке `api/tests`. Добавьте следующие строки, которые объявляют множество методов для класса **InitialTests**. Файл кода для примера включен в папку `restful_python_chapter_08_01`.

```python
def test_create_duplicated_category(self):
    """
    Убедитесь, что мы не можем создать дубликат Category
    """
    create_user_response = self.create_user(self.test_user_name,
        self.test_user_password)
    self.assertEqual(create_user_response.status_code,
        status.HTTP_201_CREATED)
    new_category_name = 'New Information'
    post_response = self.create_category(new_category_name)
    self.assertEqual(post_response.status_code, status.HTTP_201_CREATED)
    self.assertEqual(Category.query.count(), 1)
    post_response_data = json.loads(post_response.get_data(as_text=True))
    self.assertEqual(post_response_data['name'], new_category_name)
    second_post_response = self.create_category(new_category_name)
    self.assertEqual(second_post_response.status_code,
        status.HTTP_400_BAD_REQUEST)
    self.assertEqual(Category.query.count(), 1)

def test_retrieve_categories_list(self):
    """
    Убедитесь, что мы можем получить список категорий
    """
    create_user_response = self.create_user(self.test_user_name,
        self.test_user_password)
    self.assertEqual(create_user_response.status_code,
        status.HTTP_201_CREATED)
    new_category_name_1 = 'Error'
    post_response_1 = self.create_category(new_category_name_1)
    self.assertEqual(post_response_1.status_code, status.HTTP_201_CREATED)
    new_category_name_2 = 'Warning'
    post_response_2 = self.create_category(new_category_name_2)
    self.assertEqual(post_response_2.status_code, status.HTTP_201_CREATED)
    url = url_for('api.categorylistresource', _external=True)
    get_response = self.test_client.get(
        url,
        headers=self.get_authentication_headers(self.test_user_name,
            self.test_user_password))
    get_response_data = json.loads(get_response.get_data(as_text=True))
    self.assertEqual(get_response.status_code, status.HTTP_200_OK)
    self.assertEqual(len(get_response_data), 2)
    self.assertEqual(get_response_data[0]['name'], new_category_name_1)
    self.assertEqual(get_response_data[1]['name'], new_category_name_2)

def test_update_game_category(self):
    """
    Убедитесь, что мы можем обновить название для существующей категории
    """
    create_user_response = self.create_user(self.test_user_name,
        self.test_user_password)
    self.assertEqual(create_user_response.status_code,
        status.HTTP_201_CREATED)
    new_category_name_1 = 'Error 1'
    post_response_1 = self.create_category(new_category_name_1)
    self.assertEqual(post_response_1.status_code, status.HTTP_201_CREATED)
    post_response_data_1 = json.loads(post_response_1.get_data(as_text=True))
    new_category_url = post_response_data_1['url']
    new_category_name_2 = 'Error 2'
    data = {'name': new_category_name_2}
    patch_response = self.test_client.patch(
        new_category_url,
        headers=self.get_authentication_headers(self.test_user_name,
            self.test_user_password),
        data=json.dumps(data))
    self.assertEqual(patch_response.status_code, status.HTTP_200_OK)
    get_response = self.test_client.get(
        new_category_url,
        headers=self.get_authentication_headers(self.test_user_name,
            self.test_user_password))
    get_response_data = json.loads(get_response.get_data(as_text=True))
    self.assertEqual(get_response.status_code, status.HTTP_200_OK)
    self.assertEqual(get_response_data['name'], new_category_name_2)
```

В классе объявлены следующие методы, имена которых начинаются с префикса **test\_**:

* **test\_create\_duplicated\_category**: Проверяет, не позволяют ли уникальные ограничения создать две категории с одинаковыми именами или нет. Во второй раз, когда мы составим и отправим запрос HTTP **POST** с повторяющимся именем категории, мы должны получить код состояния HTTP **400 Bad Request** (`status.HTTP_400_BAD_REQUEST`), а общее количество объектов категории, извлеченных из базы данных, должно быть равно 1.
* **test\_retrieve\_categories\_list**: Проверяет, можем ли мы получить список категорий или нет. Сначала метод создает две категории, а затем убеждается, что полученный список включает две созданные категории.
* **test\_update\_game\_category**: проверяет, можем ли мы обновить одно поле для категории, в частности, поле ее имени. Код гарантирует, что имя было обновлено.

{% hint style="info" %}
**СОВЕТ**

Обратите внимание, что каждый тест, требующий определенного условия в базе данных, должен выполнять весь необходимый код, чтобы база данных находилась в этом конкретном состоянии. Например, чтобы обновить существующую категорию, сначала мы должны создать новую категорию, а затем обновить ее. Каждый тестовый метод будет выполняться без данных из ранее выполненных тестовых методов в базе данных, то есть каждый тест будет выполняться с очищенной от данных предыдущих тестов базой данных.
{% endhint %}

## Запуск юнит-тестов с nose2 и проверка покрытия тестами

Теперь запустите следующую команду, чтобы создать все необходимые таблицы в нашей тестовой базе данных, и используйте запущенный тест **nose2** для выполнения всех созданных нами тестов. Средство запуска тестов выполнит все методы нашего класса **InitialTests**, которые начинаются с префикса **test\_**, и отобразит результаты.

{% hint style="info" %}
**СОВЕТ**

Тесты не будут вносить изменения в базу данных, которую мы использовали при работе над API. Помните, что мы настроили базу данных **test\_messages** в качестве нашей тестовой базы данных.
{% endhint %}

Удалите файл **api.py**, который мы создали в предыдущей главе, из папки **api**, потому что мы не хотим, чтобы покрытие тестами учитывало этот файл. Перейдите в папку **api** и выполните следующую команду в той же виртуальной среде, которую мы использовали. Мы воспользуемся опцией **-v**, чтобы указать **nose2** печатать имена и статусы тестовых случаев. Параметр **--with-coverage** включает генерацию отчетов о тестовом покрытии:

```bash
nose2 -v --with-coverage
```

В следующих строках показан образец вывода.

```bash
test_create_and_retrieve_category (test_views.InitialTests) ... ok
test_create_duplicated_category (test_views.InitialTests) ... ok
test_request_without_authentication (test_views.InitialTests) ... ok
test_retrieve_categories_list (test_views.InitialTests) ... ok
test_update_category (test_views.InitialTests) ... ok
--------------------------------------------------------
Ran 5 tests in 3.973s
OK
----------- coverage: platform win32, python 3.5.2-final-0 --
Name                 Stmts   Miss  Cover
-----------------------------------------
app.py                   9      0   100%
config.py               11     11     0%
helpers.py              23     18    22%
migrate.py               9      9     0%
models.py              101     27    73%
run.py                   4      4     0%
status.py               56      5    91%
test_config.py          12      0   100%
tests\test_views.py     96      0   100%
views.py               204    109    47%
-----------------------------------------
TOTAL                  525    183    65%
```

По умолчанию **nose2** ищет модули, имена которых начинаются с префикса **test**. В этом случае единственным модулем, соответствующим критериям, является модуль **test\_views**. В модулях, которые соответствуют критериям, **nose2** загружает тесты из всех подклассов `unittest.TestCase` и функций, имена которых начинаются с префикса **test**.

Выходные данные содержат сведения о том, что программа запуска тестов обнаружила и выполнила пять тестов, и все они прошли успешно. В выходных данных отображается имя метода и имя класса для каждого метода в классе **InitialTests**, который начинается с префикса **test\_** и представляет тест, который необходимо выполнить.

Отчет об измерении покрытия тестового кода, предоставляемый пакетом **coverage**, использует инструменты анализа кода и обработчики трассировки, включенные в стандартную библиотеку Python, чтобы определить, какие строки кода являются исполняемыми и были выполнены. В отчете представлена таблица со следующими столбцами:

* **Name**: имя модуля Python.
* **Stmts**: количество исполняемых операторов для модуля Python.
* **Miss**: Количество пропущенных исполняемых операторов, то есть тех, которые не были выполнены.
* **Cover**: покрытие исполняемых операторов, выраженное в процентах.

У нас определенно очень низкий охват **views.py** и **helpers.py**, судя по измерениям, показанным в отчете. На самом деле, мы просто написали несколько тестов, связанных с категориями и пользователями, и поэтому вполне логично, что охват просмотров действительно низкий. Мы не создавали тесты, связанные с сообщениями.

Мы можем запустить команду покрытия с параметром командной строки **-m**, чтобы отобразить номера строк отсутствующих операторов в новом столбце **Missing**:

```bash
coverage report -m
```

Команда будет использовать информацию из последнего выполнения и отобразит отсутствующие операторы. Следующие строки показывают пример вывода, который соответствует предыдущему выполнению модульных тестов:

```bash
Name                    Stmts   Miss   Cover   Missing
---------------------------------------------------
app.py                      9      0    100%
config.py                  11     11      0%   7-20
helpers.py                 23     18     22%   13-19, 23-44
migrate.py                  9      9      0%   7-19
models.py                 101     27     73%   28-29, 44, 46, 48, 50, 52, 54,
73-75, 79-86, 103, 127-137
run.py                      4      4      0%   7-14
status.py                  56      5     91%   2, 6, 10, 14, 18
test_config.py             12      0    100%
tests\test_views.py        96      0    100%
views.py                  204    109     47%   43-45, 51-58, 63-64, 67, 71-72,
83-87, 92-94, 97-124, 127-135, 140-147, 150-181, 194-195, 198, 205-206, 209-
212, 215-223, 235-236, 239, 250-253
---------------------------------------------------
TOTAL                     525    183     65%
```

Теперь выполните следующую команду, чтобы получить аннотированные списки **HTML** с подробным описанием пропущенных строк:

```bash
coverage html
```

Откройте HTML-файл **index.html**, сгенерированный в папке **htmlcov**, в веб-браузере. На следующем рисунке показан пример отчета о покрытии, сгенерированного в формате **HTML**:

![](../../.gitbook/assets/coverage\_1.png)

Щелкните или коснитесь **views.py**, и веб-браузер отобразит веб-страницу, на которой отображаются выполненные операторы, отсутствующие и исключенные операторы, окрашенные в разные цвета. Мы можем щелкнуть или коснуться кнопок **run**, **missing** и **excluded**, чтобы показать или скрыть цвет фона, который представляет статус для каждой строки кода. По умолчанию недостающие строки кода будут отображаться на розовом фоне. Таким образом, мы должны написать модульные тесты, нацеленные на эти строки кода, чтобы улучшить наше тестовое покрытие:

![](../../.gitbook/assets/coverage\_2.png)

## Улучшение охвата тестированием

Теперь мы напишем дополнительные модульные тесты, чтобы улучшить охват тестированием. В частности, мы напишем модульные тесты, связанные с сообщениями и пользователями. Откройте существующий файл `api/tests/test_views.py` и вставьте следующие строки после последней строки в классе **InitialTests**. Нам нужен новый оператор **import**, и мы объявим новый класс **PlayerTests**. Файл кода для примера включен в папку `restful_python_chapter_08_02`:

```python
def create_message(self, message, duration, category):
    url = url_for('api.messagelistresource', _external=True)
    data = {'message': message, 'duration': duration, 'category': category}
    response = self.test_client.post(
        url,
        headers=self.get_authentication_headers(self.test_user_name,
        self.test_user_password),
        data=json.dumps(data))
    return response

def test_create_and_retrieve_message(self):
    """
    Убедитесь, что мы можем создать новое сообщение, а затем получить его
    """
    create_user_response = self.create_user(self.test_user_name,
        self.test_user_password)
    self.assertEqual(create_user_response.status_code,
        status.HTTP_201_CREATED)
    new_message_message = 'Welcome to the IoT world'
    new_message_category = 'Information'
    post_response = self.create_message(new_message_message, 15,
        new_message_category)
    self.assertEqual(post_response.status_code, status.HTTP_201_CREATED)
    self.assertEqual(Message.query.count(), 1)
    # Сообщение должно было создать новую категорию
    self.assertEqual(Category.query.count(), 1)
    post_response_data = json.loads(post_response.get_data(as_text=True))
    self.assertEqual(post_response_data['message'], new_message_message)
    new_message_url = post_response_data['url']
    get_response = self.test_client.get(
        new_message_url,
        headers=self.get_authentication_headers(self.test_user_name,
        self.test_user_password))
    get_response_data = json.loads(get_response.get_data(as_text=True))
    self.assertEqual(get_response.status_code, status.HTTP_200_OK)
    self.assertEqual(get_response_data['message'], new_message_message)
    self.assertEqual(get_response_data['category']['name'],
        new_message_category)

def test_create_duplicated_message(self):
    """
    Убедитесь, что мы не можем создать дубликат Message
    """
    create_user_response = self.create_user(self.test_user_name,
        self.test_user_password)
    self.assertEqual(create_user_response.status_code,
        status.HTTP_201_CREATED)
    new_message_message = 'Welcome to the IoT world'
    new_message_category = 'Information'
    post_response = self.create_message(new_message_message, 15,
        new_message_category)
    self.assertEqual(post_response.status_code, status.HTTP_201_CREATED)
    self.assertEqual(Message.query.count(), 1)
    post_response_data = json.loads(post_response.get_data(as_text=True))
    self.assertEqual(post_response_data['message'], new_message_message)
    new_message_url = post_response_data['url']
    get_response = self.test_client.get(
        new_message_url,
        headers=self.get_authentication_headers(self.test_user_name,
        self.test_user_password))
    get_response_data = json.loads(get_response.get_data(as_text=True))
    self.assertEqual(get_response.status_code, status.HTTP_200_OK)
    self.assertEqual(get_response_data['message'], new_message_message)
    self.assertEqual(get_response_data['category']['name'],
        new_message_category)
    second_post_response = self.create_message(new_message_message, 15,
        new_message_category)
    self.assertEqual(second_post_response.status_code,
        status.HTTP_400_BAD_REQUEST)
    self.assertEqual(Message.query.count(), 1)
```

Предыдущий код добавляет множество методов в класс **InitialTests**. Метод **create\_message** получает в качестве аргументов желаемое сообщение **message**, продолжительность **duration** и категорию **category** (имя категории) для нового сообщения. Метод создает URL-адрес и словарь данных для составления и отправки метода HTTP **POST**, создания нового сообщения и возврата ответа, сгенерированного этим запросом. Многие методы тестирования будут вызывать метод **create\_message** для создания сообщения, а затем составлять и отправлять другие HTTP-запросы к API.

В классе объявлены следующие методы, имена которых начинаются с префикса **test\_**:

* **test\_create\_and\_retrive\_message**: проверяет, можем ли мы создать новое сообщение, а затем получить его.
* **test\_create\_duplicated\_message**: Проверяет, не позволяют ли уникальные ограничения создать два сообщения с одним и тем же сообщением. Во второй раз, когда мы составляем и отправляем запрос HTTP **POST** с дублирующимся сообщением, мы должны получить код состояния HTTP **400 Bad Request** (`status.HTTP_400_BAD_REQUEST`), а общее количество объектов **Message**, извлеченных из базы данных, должно быть 1.

Откройте существующий файл `api/tests/test_views.py` и вставьте следующие строки после последней строки в классе **InitialTests**. Файл кода для примера включен в папку `restful_python_chapter_08_02`:

```python
def test_retrieve_messages_list(self):
    """
    Убедитесь, что мы можем получить список сообщений с разбивкой на страницы
    """
    create_user_response = self.create_user(self.test_user_name,
        self.test_user_password)
    self.assertEqual(create_user_response.status_code,
        status.HTTP_201_CREATED)
    new_message_message_1 = 'Welcome to the IoT world'
    new_message_category_1 = 'Information'
    post_response = self.create_message(new_message_message_1, 15,
        new_message_category_1)
    self.assertEqual(post_response.status_code, status.HTTP_201_CREATED)
    self.assertEqual(Message.query.count(), 1)
    new_message_message_2 = 'Initialization of the board failed'
    new_message_category_2 = 'Error'
    post_response = self.create_message(new_message_message_2, 10,
        new_message_category_2)
    self.assertEqual(post_response.status_code, status.HTTP_201_CREATED)
    self.assertEqual(Message.query.count(), 2)
    get_first_page_url = url_for('api.messagelistresource', _external=True)
    get_first_page_response = self.test_client.get(
        get_first_page_url,
        headers=self.get_authentication_headers(self.test_user_name,
        self.test_user_password))
    get_first_page_response_data =
        json.loads(get_first_page_response.get_data(as_text=True))
    self.assertEqual(get_first_page_response.status_code,
        status.HTTP_200_OK)
    self.assertEqual(get_first_page_response_data['count'], 2)
    self.assertIsNone(get_first_page_response_data['previous'])
    self.assertIsNone(get_first_page_response_data['next'])
    self.assertIsNotNone(get_first_page_response_data['results'])
    self.assertEqual(len(get_first_page_response_data['results']), 2)
    self.assertEqual(get_first_page_response_data['results'][0]['message'],
        new_message_message_1)
    self.assertEqual(get_first_page_response_data['results'][1]['message'],
        new_message_message_2)
    get_second_page_url = url_for('api.messagelistresource', page=2)
    get_second_page_response = self.test_client.get(
        get_second_page_url,
        headers=self.get_authentication_headers(self.test_user_name,
        self.test_user_password))
    get_second_page_response_data =
        json.loads(get_second_page_response.get_data(as_text=True))
    self.assertEqual(get_second_page_response.status_code,
        status.HTTP_200_OK)
    self.assertIsNotNone(get_second_page_response_data['previous'])
    self.assertEqual(get_second_page_response_data['previous'],
        url_for('api.messagelistresource', page=1))
    self.assertIsNone(get_second_page_response_data['next'])
    self.assertIsNotNone(get_second_page_response_data['results'])
    self.assertEqual(len(get_second_page_response_data['results']), 0)
```

В предыдущем коде к классу **InitialTests** был добавлен метод **test\_retrive\_messages\_list**. Этот метод проверяет, можем ли мы получить список сообщений с разбивкой на страницы. Сначала метод создает два сообщения, а затем убеждается, что полученный список включает два созданных сообщения на первой странице. Кроме того, этот метод гарантирует, что вторая страница не содержит никаких сообщений и что значение для предыдущей страницы включает URL-адрес первой страницы.

Откройте существующий файл `api/tests/test_views.py` и вставьте следующие строки после последней строки в классе **InitialTests**. Файл кода для примера включен в папку `restful_python_chapter_08_02`:

```python
def test_update_message(self):
    """
    Убедитесь, что мы можем обновить одно поле для существующего сообщения
    """
    create_user_response = self.create_user(self.test_user_name,
        self.test_user_password)
    self.assertEqual(create_user_response.status_code,
        status.HTTP_201_CREATED)
    new_message_message_1 = 'Welcome to the IoT world'
    new_message_category_1 = 'Information'
    post_response = self.create_message(new_message_message_1, 30,
        new_message_category_1)
    self.assertEqual(post_response.status_code, status.HTTP_201_CREATED)
    self.assertEqual(Message.query.count(), 1)
    post_response_data = json.loads(post_response.get_data(as_text=True))
    new_message_url = post_response_data['url']
    new_printed_times = 1
    new_printed_once = True
    data = {'printed_times': new_printed_times, 'printed_once': new_printed_once}
    patch_response = self.test_client.patch(
        new_message_url,
        headers=self.get_authentication_headers(self.test_user_name,
            self.test_user_password),
        data=json.dumps(data))
    self.assertEqual(patch_response.status_code, status.HTTP_200_OK)
    get_response = self.test_client.get(
        new_message_url,
        headers=self.get_authentication_headers(self.test_user_name,
            self.test_user_password))
    get_response_data = json.loads(get_response.get_data(as_text=True))
    self.assertEqual(get_response.status_code, status.HTTP_200_OK)
    self.assertEqual(get_response_data['printed_times'], new_printed_times)
    self.assertEqual(get_response_data['printed_once'], new_printed_once)

def test_create_and_retrieve_user(self):
    """
    Убедитесь, что мы можем создать нового пользователя, а затем получить его
    """
    new_user_name = self.test_user_name
    new_user_password = self.test_user_password
    post_response = self.create_user(new_user_name, new_user_password)
    self.assertEqual(post_response.status_code, status.HTTP_201_CREATED)
    self.assertEqual(User.query.count(), 1)
    post_response_data = json.loads(post_response.get_data(as_text=True))
    self.assertEqual(post_response_data['name'], new_user_name)
    new_user_url = post_response_data['url']
    get_response = self.test_client.get(
        new_user_url,
        headers=self.get_authentication_headers(self.test_user_name,
            self.test_user_password))
    get_response_data = json.loads(get_response.get_data(as_text=True))
    self.assertEqual(get_response.status_code, status.HTTP_200_OK)
    self.assertEqual(get_response_data['name'], new_user_name)
```

* Предыдущий код добавил следующие два метода в класс **InitialTests** **test\_update\_message**, который проверяет можем ли мы обновить более одного поля для сообщения, в частности, значения для полей **print\_times** и **print\_once**. Код гарантирует, что оба поля были обновлены.
* **test\_create\_and\_retrive\_user**: проверяет, можем ли мы создать нового пользователя, а затем получить его.

Мы только что закодировали несколько тестов, связанных с сообщениями, и один тест, связанный с пользователями, чтобы улучшить покрытие тестами и заметить влияние на отчет о покрытии тестами.

Теперь запустите следующую команду в той же виртуальной среде, которую мы использовали:

```bash
nose2 -v --with-coverage
```

Следующие строки показывают образец вывода:

```bash
test_create_and_retrieve_category (test_views.InitialTests) ... ok
test_create_and_retrieve_message (test_views.InitialTests) ... ok
test_create_and_retrieve_user (test_views.InitialTests) ... ok
test_create_duplicated_category (test_views.InitialTests) ... ok
test_create_duplicated_message (test_views.InitialTests) ... ok
test_request_without_authentication (test_views.InitialTests) ... ok
test_retrieve_categories_list (test_views.InitialTests) ... ok
test_retrieve_messages_list (test_views.InitialTests) ... ok
test_update_category (test_views.InitialTests) ... ok
test_update_message (test_views.InitialTests) ... ok
------------------------------------------------------------------
Ran 10 tests in 25.938s
OK
----------- coverage: platform win32, python 3.5.2-final-0 -------
Name                 Stmts   Miss  Cover
-----------------------------------------
app.py                   9      0   100%
config.py               11     11     0%
helpers.py              23      1    96%
migrate.py               9      9     0%
models.py              101     11    89%
run.py                   4      4     0%
status.py               56      5    91%
test_config.py          16      0   100%
tests\test_views.py    203      0   100%
views.py               204     66    68%
-----------------------------------------
TOTAL                  636    107    83%
```

В выходных данных были указаны сведения о том, что программа запуска тестов выполнила 10 тестов, и все они прошли успешно. Отчет об измерении покрытия тестового кода, предоставленный пакетом покрытия, увеличил процент покрытия модуля **views.py** с 47 % в предыдущем запуске до 68 %. Кроме того, процент модуля **helpers.py** увеличился с 22% до 96%, потому что мы писали тесты, в которых использовалась нумерация страниц. Новые дополнительные тесты, которые мы написали, выполняли дополнительный код в разных модулях, поэтому в отчете о покрытии есть изменения.

{% hint style="info" %}
**СОВЕТ**

Мы только что создали несколько модульных тестов, чтобы понять, как мы можем их кодировать. Однако, конечно, потребуется написать больше тестов, чтобы обеспечить надлежащее покрытие всех рекомендуемых сценариев и сценариев выполнения, включенных в API.
{% endhint %}

## Понимание стратегий развертывания и масштабируемости

**Flask** — это легкий микрофреймворк для Интернета. Однако, как и в случае с **Django**, одним из самых больших недостатков, связанных с **Flask** и **Flask-RESTful**, является _**блокировка каждого HTTP-запроса**_. Таким образом, всякий раз, когда сервер Flask получает HTTP-запрос, он не начинает работать с любыми другими HTTP-запросами во входящей очереди, пока сервер не отправит ответ на первый полученный HTTP-запрос.

Мы использовали **Flask** для разработки веб-службы **RESTful**. Основное преимущество таких веб-служб заключается в том, что они не имеют состояния, то есть они не должны сохранять состояние клиента на каком-либо сервере. Наш API — хороший пример веб-службы RESTful без сохранения состояния с **Flask** и **Flask-RESTful**. Таким образом, мы можем заставить API работать на стольких серверах, сколько необходимо для достижения наших целей масштабируемости. Очевидно, мы должны учитывать, что мы можем легко превратить сервер базы данных в наше узкое место масштабируемости.

{% hint style="info" %}
**СОВЕТ**

В настоящее время у нас есть огромное количество облачных альтернатив для развертывания веб-службы **RESTful**, которая использует **Flask** и **Flask-RESTful** и делает ее чрезвычайно масштабируемой.
{% endhint %}

Мы всегда должны убедиться, что мы профилируем API и базу данных, прежде чем развертывать первую версию нашего API. Очень важно убедиться, что сгенерированные запросы правильно выполняются в базовой базе данных и что наиболее популярные запросы не заканчиваются последовательным сканированием. Обычно необходимо добавить соответствующие индексы к таблицам в базе данных.

Мы использовали базовую HTTP-аутентификацию. Мы можем улучшить его с помощью аутентификации на основе токенов. Мы должны убедиться, что API работает под HTTPS в производственных средах. Кроме того, мы должны убедиться, что изменили следующую строку в файле `api/config.py`:

```python
DEBUG = True
```

Мы всегда должны отключать режим отладки в продакшене, и поэтому мы должны заменить предыдущую строку на следующую:

```python
DEBUG = False
```

{% hint style="info" %}
**СОВЕТ**

Для производства удобно использовать другой файл конфигурации. Однако другой подход, который становится чрезвычайно популярным, особенно для облачных приложений, заключается в хранении конфигурации в среде. Если мы хотим развернуть собственные облачные веб-службы **RESTful** и следовать рекомендациям, установленным в приложении с двенадцатью факторами, мы должны хранить конфигурацию в среде окружения.
{% endhint %}

Каждая платформа включает подробные инструкции по развертыванию нашего приложения. Все они потребуют от нас создания файла **requirements.txt**, в котором перечислены зависимости приложения вместе с их версиями. Таким образом, платформы смогут установить все необходимые зависимости, перечисленные в файле.

Запустите следующую команду `pip freeze`, чтобы сгенерировать файл **requirements.txt**.

```bash
pip freeze > requirements.txt
```

В следующих строках показано содержимое образца сгенерированного файла **requirements.txt**. Однако имейте в виду, что многие пакеты быстро увеличивают свой номер версии, и вы можете увидеть разные версии в своей конфигурации:

```python
alembic==0.8.8
aniso8601==1.1.0
click==6.6
cov-core==1.15.0
coverage==4.2
Flask==0.11.1
Flask-HTTPAuth==3.2.1
flask-marshmallow==0.7.0
Flask-Migrate==2.0.0
Flask-RESTful==0.3.5
Flask-Script==2.0.5
Flask-SQLAlchemy==2.1
itsdangerous==0.24
Jinja2==2.8
Mako==1.0.4
MarkupSafe==0.23
marshmallow==2.10.2
marshmallow-sqlalchemy==0.10.0
nose2==0.6.5
passlib==1.6.5
psycopg2==2.6.2
python-dateutil==2.5.3
python-editor==1.0.1
pytz==2016.6.1
six==1.10.0
SQLAlchemy==1.0.15
Werkzeug==0.11.11
```

## Проверьте свои знания

1. По умолчанию **nose2** ищет модули, имена которых начинаются со следующего префикса:
   1. test
   2. run
   3. unittest
2. По умолчанию **nose2** загружает тесты из всех подклассов следующего класса:
   1. unittest.Test
   2. unittest.TestCase
   3. unittest.RunTest
3. Метод **setUp** в подклассе `unittest.TestCase`:
   1. Выполняется перед запуском каждого тестового метода.
   2. Выполняется только один раз перед началом выполнения всех тестов.
   3. Выполняется только один раз после завершения выполнения всех тестов.
4. Метод **tearDown** в подклассе `unittest.TestCase`:
   1. Выполняется после запуска каждого тестового метода.
   2. Выполняется перед запуском каждого тестового метода.
   3. Выполняется после тестового метода только в случае его сбоя.
5. Если мы объявим метод **get\_accept\_content\_type\_headers** в подклассе `unittest.TestCase`, по умолчанию это будет **nose2**:
   1. Загрузит этот метод в качестве теста.
   2. Загрузит этот метод как метод **setUp** для каждого теста.
   3. Не будет загружать этот метод в качестве теста.

## Резюме

В этой главе мы настроим среду тестирования. Мы установили **nose2**, чтобы упростить обнаружение и выполнение модульных тестов, и создали новую базу данных, которая будет использоваться для тестирования. Мы написали первый раунд модульных тестов, измерили тестовое покрытие, а затем написали дополнительные модульные тесты для улучшения тестового покрытия. Наконец, мы поняли многие аспекты развертывания и масштабируемости.

Теперь, когда мы создали сложный API с помощью **Flask** в сочетании с **Flask RESTful** и протестировали его, мы перейдем к другой популярной веб-инфраструктуре Python, **Tornado**, которую мы собираемся обсудить в следующей главе.
