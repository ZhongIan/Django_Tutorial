
http://www.liujiangblog.com/course/django/89

[tutorial01](https://docs.djangoproject.com/zh-hans/2.2/intro/tutorial01/)

[tutorial02](https://docs.djangoproject.com/zh-hans/2.2/intro/tutorial02/)

# tutorial01 請求與響應

## Django 版本

    python -m django --version

    2.2

## 創建項目

    django-admin startproject mysite

## 創建應用

    python manage.py startapp polls


polls/views.py 加入以下

```py
from django.http import HttpResponse


def index(request):
    return HttpResponse("Hello, world. You're at the polls index.")
```

# tutorial02 模型與管理後台

## 數據庫配置

mysite/settings.py

## 地區、時區

```py
LANGUAGE_CODE = 'zh-hant'
TIME_ZONE = 'Asia/Taipei'
```

## INSTALLED_APPS 

默認包括了以下 Django 的自帶應用： 

* django.contrib.admin -- 管理員站點
* django.contrib.auth -- 認證授權系統
* django.contrib.contenttypes -- 內容類型框架
* django.contrib.sessions -- 會話框架
* django.contrib.messages -- 消息框架
* django.contrib.staticfiles -- 管理靜態文件的框架

## models

polls/models.py 

```py
from django.db import models

# Create your models here.
class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField('date published')


class Choice(models.Model):
    question = models.ForeignKey(Question, on_delete=models.CASCADE)
    choice_text = models.CharField(max_length=200)
    votes = models.IntegerField(default=0)
```

* Field
* ForeignKey 

為了在我們的工程中包含這個應用，我們需要在配置類 INSTALLED_APPS 中添加設置。因為 PollsConfig 類寫在文件 polls/apps.py 中，所以它的點式路徑是 'polls.apps.PollsConfig'。在文件 mysite/settings.py 中 INSTALLED_APPS 子項添加點式路徑後

```py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'polls.apps.PollsConfig',      # <---
]
```

    python manage.py makemigrations polls

通過運行 makemigrations 命令，Django 會檢測你對模型文件的修改

你可以閱讀一下你模型的遷移數據，它被儲存在 polls/migrations/0001_initial.p

讓我們看看遷移命令會執行哪些 SQL 語句。 sqlmigrate 命令接收一個遷移的名稱，然後返回對應的 SQL

    python manage.py sqlmigrate polls 0001

## DATABASES

默認值 os.path.join(BASE_DIR, 'db.sqlite3') 將會把數據庫文件儲存在項目的根目錄

    python manage.py migrate

這個 migrate 命令檢查 INSTALLED_APPS 設置，為其中的每個應用創建需要的數據表

在 makemigrations 後 python manage.py migrate

### Mysql

// settings.py

```py
import pymysql         # 一定要添加这两行！通过pip install pymysql！
pymysql.install_as_MySQLdb()

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'mysite',
        'HOST': '192.168.1.1',
        'USER': 'root',
        'PASSWORD': 'pwd',
        'PORT': '3306',
    }
}
```

在使用非SQLite的數據庫時，請務必預先在數據庫管理系統的提示符交互模式下創建數據庫，你可以使用命令：CREATE DATABASE database_name;。 

Django不會自動幫你做這一步工作。 確保你在settings文件中提供的數據庫用戶具有創建數據庫表的權限


## shell

    python manage.py shell

```py
from polls.models import Choice, Question

from django.utils import timezone
q = Question(question_text="What's new?", pub_date=timezone.now())
q.save()

Question.objects.all()
# <QuerySet [<Question: Question object (1)>]>
```
<Question: Question object (1)> 對於我們了解這個對象的細節沒什麼幫助

上面的<Question: Question object>是一個不可讀的內容展示，你無法從中獲得任何直觀的信息，為此我們需要一點小技巧，讓Django在打印對象時顯示一些我們指定的信息。 

// polls/models.py文件，修改一下question和Choice這兩個類，代碼如下：

```py
from django.db import models

class Question(models.Model):
    # ...
    def __str__(self):
        return self.question_text

class Choice(models.Model):
    # ...
    def __str__(self):
        return self.choice_text
```

另外，這裡我們自定義一個模型的方法，用於判斷問卷是否最近時間段內發布度的：

```py
import datetime

from django.db import models
from django.utils import timezone


class Question(models.Model):
    # ...
    def was_published_recently(self):
        return self.pub_date >= timezone.now() - datetime.timedelta(days=1)
```

重新啟動一個新的python shell，再來看看其他的API

    python manage.py shell

```py
from polls.models import Question, Choice

# __str__()
Question.objects.all()
#<QuerySet [<Question: What's up?>]>

# 查询API
Question.objects.filter(id=1)
#<QuerySet [<Question: What's up?>]>
Question.objects.filter(question_text__startswith='What')
#<QuerySet [<Question: What's up?>]>

# 獲取今年發布的問卷
from django.utils import timezone
current_year = timezone.now().year
Question.objects.get(pub_date__year=current_year)
#<Question: What's up?>



# 自定義方法  was_published_recently()
q = Question.objects.get(pk=1)
q.was_published_recently()
#True

# 主键查询
q = Question.objects.get(pk=1)
q.choice_set.all()

# add 3 choice
q.choice_set.create(choice_text='Not much', votes=0)
#<Choice: Not much>
q.choice_set.create(choice_text='The sky', votes=0)
#<Choice: The sky>
c = q.choice_set.create(choice_text='Just hacking again', votes=0)

# Choice對象可通過API訪問和他們關聯的Question對象
c.question
#<Question: What's up?>

# 同樣的，Question對像也可通過API訪問關聯的Choice對象
q.choice_set.all()
#<QuerySet [<Choice: Not much>, <Choice: The sky>, <Choice: Just hacking again>]>
q.choice_set.count()
#3

# API會自動進行連表操作，通過雙下劃線分割關係對象。連表操作可以無限多級，一層一層的連接。 
# 下面是查詢所有的Choices，它所對應的Question的發布日期是今年。 （重用了上面的current_year結果）
Choice.objects.filter(question__pub_date__year=current_year)
#<QuerySet [<Choice: Not much>, <Choice: The sky>, <Choice: Just hacking again>]>

# delete
c = q.choice_set.filter(choice_text__startswith='Just hacking')
c.delete()
```

## admin後台

    python manage.py createsuperuser

Username (leave blank to use 'po390'): po39086m
Email address: po39086m@gmail.com
Password: zxc

### 在admin中註冊投票應用

打開polls/admin.py文件，加入下面的內容

```py
from django.contrib import admin
from .models import Question

admin.site.register(Question)
```

# tutorial03 視圖和模板

## url-templates-view

### polls/urls

// polls/urls.py

```py
from django.urls import path

from . import views

urlpatterns = [
    # ex: /polls/
    path('', views.index, name='index'),
    # ex: /polls/5/
    path('<int:question_id>/', views.detail, name='detail'),
    # ex: /polls/5/results/
    path('<int:question_id>/results/', views.results, name='results'),
    # ex: /polls/5/vote/
    path('<int:question_id>/vote/', views.vote, name='vote'),
]
```

### polls/templates

// polls/templates/polls/index.html

```html
{% if latest_question_list %}
    <ul>
    {% for question in latest_question_list %}
        <li><a href="/polls/{{ question.id }}/">{{ question.question_text }}</a></li>
    {% endfor %}
    </ul>
{% else %}
    <p>No polls are available.</p>
{% endif %}
```

### polls/view

// polls/view

#### HttpResponse

```py
from django.http import HttpResponse
from django.template import loader

from .models import Question


def index(request):
    latest_question_list = Question.objects.order_by('-pub_date')[:5]
    template = loader.get_template('polls/index.html')
    context = {
        'latest_question_list': latest_question_list,
    }
    return HttpResponse(template.render(context, request))
```

#### 快捷方式 render

```py
from django.shortcuts import render

from .models import Question


def index(request):
    latest_question_list = Question.objects.order_by('-pub_date')[:5]
    context = {'latest_question_list': latest_question_list}
    return render(request, 'polls/index.html', context)
```

#### 返回404錯誤

// polls/views.py

如果請求的問卷ID不存在，那麼會彈出一個Http404錯誤

```py
from django.http import Http404
from django.shortcuts import render

from .models import Question
# ...
def detail(request, question_id):
    try:
        question = Question.objects.get(pk=question_id)
    except Question.DoesNotExist:
        raise Http404("Question does not exist")
    return render(request, 'polls/detail.html', {'question': question})
```

#### 快捷方式 get_object_or_404()

```py
from django.shortcuts import get_object_or_404, render

from .models import Question
# ...
def detail(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    return render(request, 'polls/detail.html', {'question': question})
```

## 使用模板系统 

### detail.html

// polls/templates/polls/polls/detail.html

```html
<h1>{{ question.question_text }}</h1>
<ul>
{% for choice in question.choice_set.all %}
    <li>{{ choice.choice_text }}</li>
{% endfor %}
</ul>
```

### 删除模板中硬编码的URLs

// ...polls/index.html

```html
- <a href="/polls/{{ question.id }}/">

+ <a href="{% url 'detail' question.id %}">
```

Django會在polls.urls文件中查找name='detail'的url，具體的就是下面這行

    path('<int:question_id>/', views.detail, name='detail'),

### URL names的命名空间

在polls/urls.py文件的開頭部分，添加一個app_name的變量來指定該應用的命名空間：

```py
from django.urls import path

from . import views

app_name = 'polls'
urlpatterns = [
    path('', views.index, name='index'),
    path('<int:question_id>/', views.detail, name='detail'),
    path('<int:question_id>/results/', views.results, name='results'),
    path('<int:question_id>/vote/', views.vote, name='vote'),
]
```

将polls/templates/polls/index.html中的

```html
- <a href="{% url 'detail' question.id %}">

+ <a href="{% url 'polls:detail' question.id %}">
```
