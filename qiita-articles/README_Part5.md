# Djangoでテスト駆動開発 その5

これはDjangoでテスト駆動開発(Test Driven Development, 以下:TDD)を理解するための学習用メモです。

参考文献は[**Test-Driven Development with Python: Obey the Testing Goat: Using Django, Selenium, and JavaScript (English Edition) 2nd Edition**](https://www.amazon.co.jp/dp/B074HXXXLS/ref=dp-kindle-redirect?_encoding=UTF8&btkr=1)を元に学習を進めていきます。

本書ではDjango1.1系とFireFoxを使って機能テスト等を実施していますが、今回はDjagno3系とGoogle Chromeで機能テストを実施していきます。また一部、個人的な改造を行っていますが(Project名をConfigに変えるなど、、)、大きな変更はありません。

⇒⇒[その1 - Chapter1](https://qiita.com/komedaoic/items/dd7abd24961250208c7c)はこちら
⇒⇒[その2 - Chapter2](https://qiita.com/komedaoic/items/58dc509e8f681da73199)はこちら
⇒⇒[その3 - Chapter3](https://qiita.com/komedaoic/items/0057b55c8ba763bda6ca)はこちら

## Part1. The Basics of TDD and Django

### Chapter5. Saving User Input: Testing the Database

ユーザーからのインプットがある場合を想定してテスト駆動開発をしていきましょう。

#### Wiring Up Our Form to Send a Post Request

ユーザーからインプットがある場合、それが問題なく保存されていることを確認するテストが必要です。
標準的なHTMLからのPOSTリクエストを想定してみます。
home.htmlにformタグを挿入して見ましょう。

```html
<!-- lists/home.html -->
<html>
    <head>
        <title>To-Do lists</title>
    </head>
    <body>
        <h1>Your To-Do list</h1>
            <form method="post">
                <input name="item_text" id="id_new_item" placeholder="Enter a to-do item">
            </form>
        <table id="id_list_table">
        </table>
    </body>
</html>

```

機能テストを実行してみましょう。

```sh
$ python functional_tests.py


DevTools listening on ws://127.0.0.1:58348/devtools/browser/2ec9655c-1dd9-4369-a97b-fb7099978b93
E
======================================================================
ERROR: test_can_start_a_list_and_retrieve_it_later (__main__.NewVisitorTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "functional_tests.py", line 42, in test_can_start_a_list_and_retrieve_it_later
    table = self.browser.find_element_by_id('id_list_table')
  File "C:--your_path--\django-TDD\venv-tdd\lib\site-packages\selenium\webdriver\remote\webdriver.py", line 360, in find_element_by_id
    return self.find_element(by=By.ID, value=id_)
  File "C:--your_path--\django-TDD\venv-tdd\lib\site-packages\selenium\webdriver\remote\webdriver.py", line 978, in find_element
    'value': value})['value']
  File "C:--your_path--\django-TDD\venv-tdd\lib\site-packages\selenium\webdriver\remote\webdriver.py", line 321, in execute
    self.error_handler.check_response(response)
  File "C:--your_path--\django-TDD\venv-tdd\lib\site-packages\selenium\webdriver\remote\errorhandler.py", line 242, in check_response
    raise exception_class(message, screen, stacktrace)
selenium.common.exceptions.NoSuchElementException: Message: no such element: Unable to locate element: {"method":"css selector","selector":"[id="id_list_table"]"}
  (Session info: chrome=79.0.3945.130)


----------------------------------------------------------------------
Ran 1 test in 9.982s

FAILED (errors=1)
```

機能テストが予想していない内容で失敗しました。
Seleniumによる実行画面を確認してみると`csrf_token`によるアクセスエラーでした。
Djangoテンプレートタグを使ってCSRFタグを追加しましょう。

```html
<!-- lists/home.html -->
<html>
    <head>
        <title>To-Do lists</title>
    </head>
    <body>
        <h1>Your To-Do list</h1>
            <form method="post">
                <input name="item_text" id="id_new_item" placeholder="Enter a to-do item">
                {% csrf_token %}
            </form>
        <table id="id_list_table">
        </table>
    </body>
</html>
```

機能テストを実施します。

```sh
DevTools listening on ws://127.0.0.1:58515/devtools/browser/0bbd1ede-a958-4371-9d8a-99b5cd55b9c3
F
======================================================================
FAIL: test_can_start_a_list_and_retrieve_it_later (__main__.NewVisitorTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "functional_tests.py", line 46, in test_can_start_a_list_and_retrieve_it_later
    "New to-do item did not appear in table"
AssertionError: False is not true : New to-do item did not appear in table

----------------------------------------------------------------------
Ran 1 test in 13.718s

FAILED (failures=1)
```

エラー内容が`New to-do item did not appear in table`となりました。
これはまだサーバー側がPOSTリクエストの処理を追加していないためです。

#### Processing a POST Request on the Server

`home_page` View にPOST処理を追加していきましょう。その前に*list/tests.py*を開いて`HomePageTest`にPOSTが保存されているかどうかを確認するテストを追加します。

```python
# lists/tests.py

from django.test import TestCase


class HomePageTest(TestCase):

    def test_users_home_template(self):
        response = self.client.get('/')  # URLの解決
        self.assertTemplateUsed(response, 'lists/home.html')

    def test_can_save_a_POST_request(self):
        response = self.client.post('/', data={'item_text': "A new list item"})
        self.assertIn('A new list item', response.content.decode())
```

ここで現在の*lists/views.py*を確認しておきます。

```python
# lists/views.py

from django.shortcuts import render


def home_page(request):
    return render(request, 'lists/home.html')
```

現在のViewはリクエストに対して*home.html*を返すのみとなっています。
単体テストを行ってみましょう。

```sh
python manage.py test

======================================================================
FAIL: test_can_save_a_POST_request (lists.tests.HomePageTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "C:--your_path--\django-TDD\lists\tests.py", line 14, in test_can_save_a_POST_request
    self.assertIn('A new list item', response.content.decode())
AssertionError: 'A new list item' not found in '<!-- lists/home.html -->\n<html>\n    <head>\n        <title>To-Do lists</title>\n    </head>\n    <body>\n        <h1>Your To-Do list</h1>\n            <form method="post">\n                <input name="item_text" id="id_new_item" placeholder="Enter a to-do item">\n                <input type="hidden" name="csrfmiddlewaretoken" value="WxS3OX6BLnYLQuT4saIKUU4O18Z9mDZDIfhIrUiBjeeizFlf2ajmbj86QyzEo04R">\n            </form>\n        <table id="id_list_table">\n        </table>\n    </body>\n</html>\n'

----------------------------------------------------------------------
```

テスト結果を確認すると`response.content.decode()`の中身が表示されているのが分かります。
中身を確認すると、`{% csrf_token %}`により`<input type="hidden">`という形でCSRFトークンがDjangoによって追加されていることがわかります。

テストをパスするため、POSTで投げられている`item_text`を返せるようにViewを変更します。

```python
# lists/views.py

from django.shortcuts import HttpResponse  # 追加
from django.shortcuts import render


def home_page(request):
    if request.method == 'POST':
        return HttpResponse(request.POST['item_text'])
    return render(request, 'lists/home.html')

```

これで単体テストはパスしましたが、これは本当に行いたいことではありません。


#### Passing Python Variables to Be Rendered in the Template

Djagnoテンプレートでは`{{  }}`を使ってViewからの変数を利用することが可能です。
これを使ってPOSTさた内容を表示できるように`home.html`に追加しておきましょう。

```html
    <body>
        <h1>Your To-Do list</h1>
            <form method="post">
                <input name="item_text" id="id_new_item" placeholder="Enter a to-do item">
                {% csrf_token %}
            </form>
        <table id="id_list_table">
            <tr><td>{{ new_item_text }}</td></tr>
        </table>
    </body>
```

Viewで`new_item_text`を返すことで`home.html`に表示させることができますが、そもそもリクエストに対して`home.html`を返しているかどうかもテストしておかなければなりません。単体テストに追加しましょう。

```python
def test_can_save_a_POST_request(self):
    response = self.client.post('/', data={'item_text': "A new list item"})
    self.assertIn('A new list item', response.content.decode())
    self.assertTemplateUsed(response, 'lists/home.html')  # 追加
```

単体テストを実行すると`AssertionError: No templates used to render the response`となりました。
これはPOST時のレスポンスにHTMLを指定していないためです。Viewを変更しましょう。

```python
def home_page(request):
    if request.method == 'POST':
        return render(request, 'lists/home.html', {
            'new_item_text': request.POST['item_text'],
        })
    return render(request, 'lists/home.html')
```

これで単体テストはパスできました。それでは機能テストを実行してみます。

```sh
$ python functional_tests.py

======================================================================
FAIL: test_can_start_a_list_and_retrieve_it_later (__main__.NewVisitorTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "functional_tests.py", line 46, in test_can_start_a_list_and_retrieve_it_later
    "New to-do item did not appear in table"
AssertionError: False is not true : New to-do item did not appear in table

----------------------------------------------------------------------
```

アイテムを追加して表示できているはずにも関わらず、`AssertionError: False is not true : New to-do item did not appear in table`となりました。
エラー内容をより詳細に確認するために、機能テストの一部を書き換えて実行してみます。

```python

self.assertTrue(
    any(row.text == "1: Buy dorayaki" for row in rows),
    f"New to-do item did not appear in table. Contents were: \
    \n{table.text}"
)
```

もう一度実行します。

```sh
$ python functional_tests.py

======================================================================
FAIL: test_can_start_a_list_and_retrieve_it_later (__main__.NewVisitorTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "functional_tests.py", line 47, in test_can_start_a_list_and_retrieve_it_later
    \n{table.text}"
AssertionError: False is not true : New to-do item did not appear in table. Contents were:
Buy dorayaki

----------------------------------------------------------------------
```

実際のtableのテキストを確認することができました。
これを見ると`"1: Buy dorayaki" `となるべきところが`Buy dorayaki`となっているためにエラーが出ているようです。

これを解決するViewはまだ後になりそうです。今回は`item_text`が返ってきているのかどうかを確認するためなので、機能テストをそのように書き換えます。

```python
# functional_tests.py
[...]
self.assertIn('1: Buy dorayaki', [row.text for row in rows])
[...]
```

機能テストを実行します。

```sh
$ python functional_tests.py

======================================================================
FAIL: test_can_start_a_list_and_retrieve_it_later (__main__.NewVisitorTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "functional_tests.py", line 50, in test_can_start_a_list_and_retrieve_it_later
    self.assertIn('1: Buy dorayaki', [row.text for row in rows])
AssertionError: '1: Buy dorayaki' not found in ['Buy dorayaki']

----------------------------------------------------------------------
Ran 1 test in 8.901s

```

やはりエラーが出てしまいました。ポイントは追加されたアイテムを数え上げることですが、
機能テストをパスする最速の方法は`1: `を追加することです。

```html
<table id="id_list_table">
    <tr><td>1: {{ new_item_text }}</td></tr>
</table>
```

これで機能テストを実行すると`self.fail("Finish the test!")`となり、機能テストは(とりあえずは)実行できました。

機能テストがパスできたので、新たに機能テストを更新します。

```python
# django-tdd/functional_tests.py

[...]

# テキストボックスは引続きアイテムを記入することができるので、
# 「どら焼きのお金を請求すること」を記入した(彼はお金にはきっちりしている)
inputbox = self.browser.find_element_by_id('id_new_item')
inputbox.send_keys("Demand payment for the dorayaki")
inputbox.send_keys(Keys.ENTER)
time.sleep(1)

# ページは再び更新され、新しいアイテムが追加されていることが確認できた
table = self.browser.find_element_by_id('id_list_table')
rows = table.find_elements_by_tag_name('tr')
self.assertIn('1: Buy dorayaki', [row.text for row in rows])
self.assertIn('2: Demand payment for the dorayaki',
[row.text for row in rows])

# のび太はこのto-doアプリが自分のアイテムをきちんと記録されているのかどうかが気になり、
# URLを確認すると、URLはのび太のために特定のURLであるらしいことがわかった
self.fail("Finish the test!")

# のび太は一度確認した特定のURLにアクセスしてみたところ、

# アイテムが保存されていたので満足して眠りについた。

[...]

```

これを実行してみましょう。

```sh
$ python functional_tests.py

======================================================================
FAIL: test_can_start_a_list_and_retrieve_it_later (__main__.NewVisitorTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "functional_tests.py", line 56, in test_can_start_a_list_and_retrieve_it_later
    self.assertIn('1: Buy dorayaki', [row.text for row in rows])
AssertionError: '1: Buy dorayaki' not found in ['1: Demand payment for the dorayaki']

----------------------------------------------------------------------
Ran 1 test in 13.815s
```

やはり`id_list_table`においてitemを数え上げられていないことが原因で機能テストがパスできないようです。

機能テストを更新したのでコミットしておきます。

```sh
$ git add .
$ git commit -m "post request returns id_list_table"
```

機能テストのテキストがあるかどうかの判断は切り分けて処理を行うのが賢いやり方です。
機能テストをリファクタリングしましょう。

```python
# functional_tests.py

[...]
def tearDown(self):
    self.browser.quit()

def check_for_row_in_list_table(self, row_text):
    table = self.browser.find_element_by_id('id_list_table')
    rows = table.find_elements_by_tag_name('tr')
    self.assertIn(row_text, [row.text for row in rows])

[...]

        # のび太がエンターを押すと、ページは更新され、
        # "1: どら焼きを買うこと"がto-doリストにアイテムとして追加されていることがわかった
        inputbox.send_keys(Keys.ENTER)
        time.sleep(1)  # ページ更新を待つ。
        self.check_for_row_in_list_table('1: Buy dorayaki')

        # テキストボックスは引続きアイテムを記入することができるので、
        # 「どら焼きのお金を請求すること」を記入した(彼はお金にはきっちりしている)
        inputbox = self.browser.find_element_by_id('id_new_item')
        inputbox.send_keys("Demand payment for the dorayaki")
        inputbox.send_keys(Keys.ENTER)
        time.sleep(1)

        # ページは再び更新され、新しいアイテムが追加されていることが確認できた
        self.check_for_row_in_list_table('2: Demand payment for the dorayaki')

[...]

```

#### The Django ORM and Our First Model

DjangoのORM(Object-Relational Mapper)をつかってItemを追加するためのテーブルを作成していきましょう。
これは*lists/models.py*に記述していきますが、先に単体テストを書いていきましょう。

```python
# lists/tests.pyに追加

from lists.models import Item


class ItemModelTest(TestCase):

    def test_saving_and_retrieving_item(self):
        first_item = Item()
        first_item.text = 'The first (ever) list item'
        first_item.save()

        second_item = Item()
        second_item.text = 'Item the second'
        second_item.save()

        saved_items = Item.objects.all()
        self.assertEqual(saved_items.count(), 2)

        first_saved_item = saved_items[0]
        second_saved_item = saved_items[1]
        self.assertEqual(first_saved_item.text, 'The first (ever) list item')
        self.assertEqual(second_saved_item.text, 'Item the second')

```

モデルを定義する際に確認する点は以下の2点のようです。

- モデルにデータが保存されているか

- モデルからデータが取り出せているか

これを単体テストで確認します。

```sh
$ python manage.py test

======================================================================
ERROR: lists.tests (unittest.loader._FailedTest)
----------------------------------------------------------------------
ImportError: Failed to import test module: lists.tests
Traceback (most recent call last):
  File "C:\Users\masayoshi\AppData\Local\Programs\Python\Python37\lib\unittest\loader.py", line 436, in _find_test_path
    module = self._get_module_from_name(name)
  File "C:\Users\masayoshi\AppData\Local\Programs\Python\Python37\lib\unittest\loader.py", line 377, in _get_module_from_name
    __import__(name)
  File "C:--your_path--\django-TDD\lists\tests.py", line 4, in <module>
    from lists.models import Item
ImportError: cannot import name 'Item' from 'lists.models' (C:--your_path--\django-TDD\lists\models.py)
```

モデルを定義していないので`ImportError`となりました。早速モデルを定義します。

```python
# lists/models.py

from django.db import models


class Item(object):
    pass
```

単体テストを実行すると結果は`AttributeError: 'Item' object has no attribute 'save'`となります。
`Item`クラスに`save`メソッドを追加するには`Model class`を継承する必要があります。
`lists/models.py`を書き換えましょう.

```python
# lists/models.py

from django.db import models


class Item(models.Model):
    pass

```

単体テストの結果は`django.db.utils.OperationalError: no such table: lists_item`となります。
これは*lists/models.py*に`Item`クラスを追加したものの、実際のテーブルは作成されていないためです。

##### Our First Database Migration

DjangoのORMを使ってマイグレーションをしましょう。

```sh
$ python manage.py makemigrations
Migrations for 'lists':
  lists\migrations\0001_initial.py
    - Create model Item
```

listsフォルダ内に`/migrations`というフォルダが作成されました。

単体テストを行うと`AttributeError: 'Item' object has no attribute 'text'`という結果になりました。

##### The Test Gets Surprisingly Far

`Item`クラスに`text`を追加していきます。

```python
# lists/models.py

from django.db import models


class Item(models.Model):
    text = models.TextField()
```

単体テストを行うと`django.db.utils.OperationalError: no such column: lists_item.text`となります。
これは作成したItemというテーブルに`text`がまだ追加されていないためです。
追加するにはマイグレーションする必要があります。

```sh
$ python manage.py makemigrations

You are trying to add a non-nullable field 'text' to item without a default; we can't do that (the database needs something to populate existing rows).
Please select a fix:
 1) Provide a one-off default now (will be set on all existing rows with a null value for this column)
 2) Quit, and let me add a default in models.py
Select an option: 2
```

`models.TextField()`にはdefault値を設定する必要があるようなので追加します。

```python
# lists/models.py

from django.db import models


class Item(models.Model):
    text = models.TextField(default='')
```

モデルを変更したのでマイグレーションします。

```sh
$ python manage.py makemigrations
Migrations for 'lists':
  lists\migrations\0002_item_text.py
    - Add field text to item
```

これで単体テストはパスできました。
コミットしておきましょう。

```
$ git add lists
$ git commit -m "Model for list Items and associated migration"
```