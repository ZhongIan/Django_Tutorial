
# 參考

[django-tutorial](https://github.com/twtrubiks/django-tutorial)

# 環境

django 2.2
python 3.6

# 目錄

- [參考](#參考)
- [環境](#環境)
- [目錄](#目錄)
- [建立專案](#建立專案)
- [建立APP](#建立app)
- [執行 runserver](#執行-runserver)
- [專案內容](#專案內容)
    - [Views](#views)
    - [templates](#templates)
    - [URLconf](#urlconf)
    - [Models](#models)
        - [Model Field.choices](#model-fieldchoices)
    - [model form db](#model-form-db)    - [Django ORM](#django-orm)
        - [Create](#create)
        - [Read](#read)
        - [Update](#update)
        - [Delete](#delete)
    - [Admin Site 後台管理](#admin-site-後台管理)
        - [註冊 model](#註冊-model)
    - [error pages](#error-pages)

# 建立專案

    django-admin startproject django_tutorial

# 建立APP

    python manage.py startapp musics

# 執行 runserver

    python manage.py runserver

# 專案內容

## Views
## templates

```html
<body>
    {{data}}
</body>
```

[django 2.2 Doc](https://docs.djangoproject.com/en/2.2/topics/templates/)

## URLconf

## Models

```py
from django.db import models


# Create your models here.
class Music(models.Model):
    song = models.TextField(default="song")
    singer = models.TextField(default="AKB48")
    last_modify_date = models.DateTimeField(auto_now=True)
    created = models.DateTimeField(auto_now_add=True)

    class Meta:
        db_table = "music"
```

default : 代表默認值，也就是如果你沒有指定的話會用默認值。

auto_now_add : 新增時會幚你自動加上建立時間。

auto_now : 資料有更新時會幚你自動加上更新的時間。

[django fields](https://docs.djangoproject.com/en/2.2/ref/models/fields/)

從 model.py 生成 SQL

    python manage.py makemigrations

    python manage.py migrate

makemigrations ： 會幚你建立一個檔案，去記錄你更新了哪些東西。

migrate ： 根據 makemigrations 建立的檔案，去更新你的 DATABASE 

### Model Field.choices

model

```py
from django.db import models

TYPE_CHOICES = (
    ('T1', 'type 1'),
    ('T2', 'type 2'),
    ('T3', 'type 3'),
    ('T4', 'type 4'),
)

# Create your models here.
class Music(models.Model):
    song = models.TextField(default="song")
    singer = models.TextField(default="AKB48")
    last_modify_date = models.DateTimeField(auto_now=True)
    created = models.DateTimeField(auto_now_add=True)
    type = models.CharField(
        max_length=2,
        choices=TYPE_CHOICES,
        default="T1"
    )

    class Meta:
        db_table = "music"

    def display_type_name(self):
        return self.get_type_display()
```

templates

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
{{ data }}
{% for music in musics %}
    <p>id : {{ music.id }}</p>
    <p>song : {{ music.song }}</p>
    <p>singer : {{ music.singer }}</p>
    <p>type : {{ music.type }}</p>
    <!-- ***********************     display_type_name      ************************* -->
    <p>display_type_name : {{ music.display_type_name }}</p>
{% endfor %}

</body>
</html>
```

## model form db

如果說現在我們已經有一個 db，需要建立 model 讓他 map 到 db

    python manage.py inspectdb > models.py

## Django ORM

記得必須先 import 你的 models

    from musics.models import Music

### Create

    Music.objects.create(song='song1', singer='SKE48')

### Read

    Music.objects.all()

    Music.objects.get(pk=3)

    Music.objects.filter(id=1)

其實大部分情況來說pk和id是一樣的，我們知道pk代表primary key的縮寫，也就是任何model中都有的主鍵，那麼id呢，大部分時候也是model的主鍵，所以在這個時候我們可以認為pk和id是完全一樣的。 但是有時候不一樣？什麼時候？是的，你猜到了，當model的主鍵不是id的時候，這種情況雖然少，但是django為我們想到了，我們來看一下

```py
class Student(model.Model):  
    my_id = models.AutoField(primary_key=True)  
    name = models.Charfield(max_length=32)  
```

[pk id 內容參考自](https://zhangfortune.iteye.com/blog/2124979)


### Update

    data=Music.objects.filter(id=1)

    data.update(song='song_update')

### Delete

    data=Music.objects.filter(id=4)

    data.delete()

## Admin Site 後台管理

建立超級使用者

    python manage.py createsuperuser

### 註冊 model

我們可以註冊 model，讓後台可以操作 database

請在 admin.py 裡面新增下方程式碼，這段程式碼只是去註冊 model 而已

```py
from django.contrib import admin

# Register your models here.
from django.contrib import admin
from musics.models import Music

admin.site.register(Music)
```

## error pages

先到 django_tutorial/settings.py 中設定幾個東西，

分別是 DEBUG 和 ALLOWED_HOSTS ( 這兩個設定是為了顯示 error pages )，

INSTALLED_APPS ( 這個則是為了要讓他找的到 template )，範例如下，

```py
DEBUG = False

ALLOWED_HOSTS = ['*']


# Application definition

INSTALLED_APPS = [
    .....
    'django_tutorial',
]

```

補充，預設為 DEBUG = True，這時候 django 會使用 standard 404 debug template，所以要記得修改。

建立 templates 資料夾，在底下建立 page_404.html 以及 page_500.html，

然後再建立一個 views 資料夾，底下建立 error_views.py，範例如下，

```py
from django.shortcuts import render


def view_404(request):
    return render(request, 'django_tutorial/error_pages/page_404.html', status=404)


def view_500(request):
    return render(request, 'django_tutorial/error_pages/page_500.html', status=500)
```

**目錄結構**

    django_tutorial
        django_tutorial
            templates
                django_tutorial
                    error_pages
                        page_404.html
                        page_500.html
            views
                error_views.py

            ...
            settings.py
            urls.py
            ...

這邊補充說明一下，前面在 INSTALLED_APPS 中設定 django_tutorial，

主要就是為了讓他可以抓到 django_tutorial/error_pages/page_404.html

最後就是在 django_tutorial/urls.py 設定 handler404 以及 handler500，

```py
handler404 = "django_tutorial.views.error_views.view_404"
handler500 = "django_tutorial.views.error_views.view_500"
```

[django 2.2 customizing-error-views](https://docs.djangoproject.com/en/2.2/topics/http/views/#customizing-error-views)