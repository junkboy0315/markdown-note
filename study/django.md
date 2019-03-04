# Django

[[toc]]

## インストール

python の仮想環境を作成

```bash
python -m venv .venv
```

`requirements.txt`の作成とインストール

```txt
Django~=2.1.5
```

```bash
pip install -r requirements.txt
```

Django の初期ファイル群を作成

```bash
django-admin.exe startproject mysite .
```

`mysite/settings.py`の内容を必要に応じて修正する

```py
# DEBUG=Trueかつ以下が空の場合、自動的にlocalhost等が設定される
ALLOWED_HOSTS = []

LANGUAGE_CODE = 'ja'
TIME_ZONE = 'Asia/Tokyo'
STATIC_ROOT = os.path.join(BASE_DIR, 'static')
```

マイグレーションを実行(デフォルトでは sqlite3 がすぐに使えるよう設定されている)

```bash
python manage.py migrate
```

サーバを起動

```bash
python manage.py runserver
```

## モデル

`blog`という名前のアプリケーションを作成

```bash
python manage.py startapp blog
```

`mysite/settings.py`にアプリケーションを追加しておく

```py
INSTALLED_APPS = [
    # ....other apps
    'blog',
]
```

`blog/models.py`にモデルを定義する（[フィールドタイプの一覧](https://docs.djangoproject.com/ja/2.0/ref/models/fields/#field-types)）

```py
from django.db import models
from django.utils import timezone


class Post(models.Model):
    author = models.ForeignKey('auth.User', on_delete=models.CASCADE)
    title = models.CharField(max_length=200)
    text = models.TextField()
    created_date = models.DateTimeField(default=timezone.now)
    published_date = models.DateTimeField(blank=True, null=True)

    def publish(self):
        self.published_date = timezone.now()
        self.save()

    def __str__(self):
        return self.title

    # 計算で算出したいプロパティは下記のように定義する
    @property
    def author_name_and_title(self):
      return self.author.name + self.title
```

マイグレーションを行う

```bash
# モデルの変更を検出し、マイグレーションファイルを作成
python manage.py makemigrations blog

# マイグレーションファイルをDBに適用
python manage.py migrate blog
```

Django admin で Post を管理できるよう、`blog/admin.py`に Post を追加する。

```py
from .models import Post
admin.site.register(Post)
```

管理者を作成する

```bash
python manage.py createsuperuser
```

`http://localhost:8000/admin`で管理画面にログインすると、ブラウザ上から DB を編集できる。

## ルーティング

`urls.py`は、ルーティングの設定ファイルである。ルートと、それに対応する処理（一般的には view）を設定する。

`mysite/urls.py`で、`/blog`に来たリクエストを`blog/urls.py`に委任する

```py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('blog/', include('blog.urls'))
]
```

`blog/urls.py`では、全ての処理を`view.post_list`に任せる。

```py
from django.urls import path

from . import views

urlpatterns = [
    path('', views.post_list, name='post_list'),
]
```

※ `name`とは？

- ビューを識別するために使われる URL の名前
- ビューと同じ名前にすることもできるし、別の名前にすることもできる
- ユニークで覚えやすいものにしておくこと

## ビュー

`blog/views.py`

```py
from django.shortcuts import render

def post_list(request):
    return render(request, 'blog/post_list.html', {})
```

テンプレートは、`アプリケーションフォルダ/templates/アプリケーション名`の中に配置すること。
アプリケーション名が重複しているのは、あとでより複雑な構成にする時に楽にするためらしい。

`blog/templates/blog/post_list.html`

```html
<div>hello world</div>
```

## Django ORM

### Django shell

Django shell の起動 (python shell に見えるが、Django も動いてるよ)

```bash
python manage.py shell
```

shell では、Model を使って様々な操作を行える。

### 一覧の取得

`model.objects.all()`

一覧（クエリセット）を取得できる。クエリセット＝モデルのインスタンスのリストのこと。

```py
from blog.models import Post
Post.objects.all()
# <QuerySet [<Post: Hello1>, <Post: Hello2>]> => これがクエリセット
```

### 条件を指定して 1 件取得

`model.objects.get()`

```py
# django標準のユーザ管理機能から、Userモデルを呼び出す
from django.contrib.auth.models import User

me = User.objects.get(username='ola') # => 単品のオブジェクトが返る
```

### 作成

`model.objects.create()`

```py
Post.objects.create(author=me, title="Sample Title", text='Test')
```

### 条件を指定して複数件取得

`model.objects.filter()`

```py
Post.objects.filter(author=me) # => クエリセットが返る
```

「含む」などの条件を指定したい場合は、`フィールド名__contains`等の形式で指定する。

```py
Post.objects.filter(title__contains='sample')
Post.objects.filter(published_date__lte=timezone.now())
```

### 並べ替え

`model.objects.order_by()`

```py
Post.objects.order_by('created_date')
Post.objects.order_by('-created_date')
```

### チェーン

```py
Post.objects.\
  filter(title__contains='sample').\
  order_by('created_date').\
  all()
```

### 参照・逆参照先のデータの取得

後続の処理で何度もアクセスされるオブジェクトを先に取得しておきたいときには、`select_related`や`prefetch_related`を使う。

[参考](https://akiyoko.hatenablog.jp/entry/2016/08/03/080941)

#### select_related

one 側のオブジェクト（Foreign Key など）を見に行くときにつかう。
INNER JOIN または LEFT OUTER JOIN されるのでクエリの回数を減らせる。

```py
# many->one
BlogPost.objects.filter(pk=1).select_related('user')
```

#### prefetch_related

one 側のオブジェクトに加え、many 側のオブジェクト群を取得することができる。
複数回のクエリを発行して Python 側で結合するので、select_related よりはクエリ回数は増える。

```py
# one->many (reverse relationship)
User.objects.filter(pk=1).prefetch_related('blogposts')

# many->many
BlogPost.objects.filter(pk=1).prefetch_related('categories')
```

#### prefetch_related するデータにフィルタを掛ける

`django.db.models.Prefetch`を使う。

```py
User.objects.all().prefetch_related(
  Prefetch(
      "blogposts",
      queryset=Blogposts.objects.filter(some_key='some_value')
  )
)
```

### 2 種類のインスタンス作成方法

```py
# こちらは`save()`は不要
MODEL_NAME.objects.create(kwargs)

# こちらは`save()`が必要
obj = MODEL_NAME(kwargs)
obj.save()
```

### Model.objects.get()の弱点

get 関数は条件にあうインスタンス（オブジェクト）が見つからなかったとき、`ObjectDoesNotExist`を raise する。これは不都合なことが多いので、通常は`get_object_or_404` 関数を使う。

```py
from django.shortcuts import get_object_or_404
get_object_or_404(Person, id=20)
```

## 動的データを表示する

view.py において、Model と Template をバインドする。

`blog/views.py`

```py
from django.shortcuts import render
from django.utils import timezone
from .models import Post

def post_list(request):
    posts = Post.objects.\
        filter(published_date__lte=timezone.now()).\
        order_by('published_date')
    return render(request, 'blog/post_list.html', {'posts': posts})
```

`blog/templates/blog/post_list.html`

```html
{% for post in posts %}
<div>
  {% if post.published_date %}
  <div class="date">{{ post.published_date }}</div>
  {% endif %}

  <h1>{{ post.title }}</h1>
  <p>{{ post.text | linebreaksbr }}</p>
</div>
{% endfor %}
```

- `{% %}`はテンプレートタグといい、構文を使うことができる
- `{ { some_key } }`で、view から渡された値を参照できる
- `linebreaksbr`は改行を段落に変換するフィルタ

## static files

`blog/static/***.css`など、アプリケーション内の`static`フォルダに置いた全てのファイルが、`/static`以下にサーブされる。

テンプレートから呼び出す場合は以下のようにする。アプリ内の`static`フォルダからの相対パスになるので注意する。

```html
{% load static %}

<link rel="stylesheet" href="{% static 'blog.css' %}" />
```

## テンプレート拡張

他のテンプレートを親として読み込むことができる

親側テンプレート(`base.html`)

```xml
<head><link rel="stylesheet" href="{% static 'blog.css' %}"></head>

<h1><a href="/">Django Girls Blog</a></h1>

{% block content %}
{% endblock %}
```

子側テンプレート（`post_list.html`）

```xml
{% extends 'blog/base.html' %}

{% block content %}
  {% for post in posts %}
    <div>
      <p>{{ post.published_date }}</p>
      <h1>{{ post.title }}</h1>
      <p>{{ post.text | linebreaksbr }}</p>
    </div>
  {% endfor %}
{% endblock %}
```

## url params の利用

`blog/urls.py`

```py
urlpatterns = [
    path('', views.post_list, name='post_list'),
    path('<int:pk>/', views.post_detail, name='post_detail'), # 追加
]
```

`blog/views.py`

```py
def post_detail(request, pk): # ここにparamsがインジェクトされる
    post = Post.objects.get(pk=pk)
    return render(request, 'blog/post_detail.html', {'post': post})
```

### リンクの動的な作成

```html
<a href="{% url 'post_detail' pk=post.pk %}"></a>
```

- `post_detail`は urlpatterns の name で指定する。これにより、実際のパスマッピングが変更されたとしても、テンプレートを修正しなくてよくなる。
- `pk=**`の部分は、本来は url param として渡るものの代わりに、値を明示的に指定している

### エラーではなく 404

オブジェクトが見つからなかった場合に、エラーページではなく 404 ページを表示する方法。

```py
# 見つからなかった場合に、エラーページが表示される
Post.objects.get(pk=pk)

# 見つからなかった場合に、404ページが表示される
from django.shortcuts import get_object_or_404
get_object_or_404(Post, pk=pk)
```

## Forms

### ModelForm の作成

モデルを基にしてフォームを作成するための仕組み＝ ModelForm

`blog/forms.py`

```py
from django import forms
from .models import Post

class PostForm(forms.ModelForm):
    class Meta:
        # どのモデルを使うか
        model = Post
        # どの項目をフォームとして表示するか
        fields = ('title', 'text')
```

### ルーティング

`blog/urls.py`

```py
urlpatterns = [
    path('<int:pk>/', views.post_detail, name='post_detail'), # データを閲覧
    path('<int:pk>/edit', views.post_edit, name='post_edit'), # 既存のデータを編集
    path('new/', views.post_new, name='post_new'), # 新規にデータを追加
]
```

`blog/views.py`

```py
from django.shortcuts import redirect

from .forms import PostForm

def post_detail(request, pk):
    post = Post.objects.get(pk=pk)
    return render(request, 'blog/post_detail.html', {'post': post})

def post_new(request):
    # フォームデータを受け取った時
    if request.method == "POST":
        # POSTされたデータをオブジェクトに変換
        form = PostForm(request.POST)

        if form.is_valid():
            # ユーザを追加したいのでコミットはまだしない
            post = form.save(commit=False)
            post.author = request.user
            post.published_date = timezone.now()
            post.save()

        # 成功したら個別ページにリダイレクトする
        return redirect('post_detail', pk=post.pk)

    # 単にGETで表示された時
    else:
        form = PostForm()
        return render(request, 'blog/post_edit.html', {'form': form})

def post_edit(request, pk):
    # 編集対象のデータを取得しておく
    post = Post.objects.get(pk=pk)

    if request.method == 'POST':
        form = PostForm(request.POST, instance=post)
        # ...ここの処理はpost_newと同じ...
    else:
        form = PostForm(instance=post)
        # ...以降の処理はpost_newと同じ...
```

### フォームの表示

`blog/templates/blog/post_edit.html`

view から受け取った`form`を使ってフォームを表示する

```xml
<h1>New post</h1>
  <form method="POST">
  {% csrf_token %}
  {{ form.as_p }}
  <button type="submit">Save</button>
</form>
```

### ハイパーリンクの作成

```xml
<!-- 新規投稿 -->
<a href="{% url 'post_new' %}">add new post</a>

<!-- 編集 -->
<a href="{% url 'post_edit' pk=*** %}">edit post</a>
```