
[tutorial04](https://docs.djangoproject.com/zh-hans/2.2/intro/tutorial04/)

[中文 04](http://www.liujiangblog.com/course/django/90)






# tutorial04 表單和類視圖

## 表單form

投票界面

// polls/detail.html

```html
<h1>{{ question.question_text }}</h1>

{% if error_message %}<p><strong>{{ error_message }}</strong></p>{% endif %}

<form action="{% url 'polls:vote' question.id %}" method="post">
{% csrf_token %}
{% for choice in question.choice_set.all %}
    <input type="radio" name="choice" id="choice{{ forloop.counter }}" value="{{ choice.id }}">
    <label for="choice{{ forloop.counter }}">{{ choice.choice_text }}</label><br>
{% endfor %}
<input type="submit" value="Vote">
</form>
```

一個包含數據choice=#的POST請求將被發送到指定的url，#是被選擇的選項的ID。這就是HTML表單的基本概念

forloop.counter是DJango模板系統專門提供的一個變量，用來表示你當前循環的次數，一般用來給循環項目添加有序數標。

POST請求 {% csrf_token %} 但ajax不能用

// polls/views.py

```py
from django.http import HttpResponse, HttpResponseRedirect
from django.shortcuts import get_object_or_404, render
from django.urls import reverse

from .models import Choice, Question
# ...
def vote(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    try:
        selected_choice = question.choice_set.get(pk=request.POST['choice'])
    except (KeyError, Choice.DoesNotExist):     
        return render(request, 'polls/detail.html', {
            'question': question,
            'error_message': "You didn't select a choice.",
        })
    else:
        selected_choice.votes += 1
        selected_choice.save()       
        return HttpResponseRedirect(reverse('polls:results', args=(question.id,)))
```

通常我們會給個默認值，防止這種異常的產生，例如request.POST['choice',None]

HttpResponseRedirect需要一個參數：重定向的URL

vote()視圖重定向到了問卷的結果顯示頁面

// polls/views.py

```py
from django.shortcuts import get_object_or_404, render

def results(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    return render(request, 'polls/results.html', {'question': question})
```

// polls/templates/polls/results.html

```html
<h1>{{ question.question_text }}</h1>

<ul>
{% for choice in question.choice_set.all %}
    <li>{{ choice.choice_text }} -- {{ choice.votes }} vote{{ choice.votes|pluralize }}</li>
{% endfor %}
</ul>

<a href="{% url 'polls:detail' question.id %}">Vote again?</a>
```

## 使用通用視圖

上面的detail、index和results視圖的代碼非常相似

Django很善解人意的幫你想辦法偷懶，於是它提供了一種快捷方式，名為“通用視圖”

* 修改URLconf設置 
* 刪除一些舊的無用的視圖 
* 採用基於類視圖的新視圖

### URLconf

// polls/urls.py

```py
from django.urls import path

from . import views

app_name = 'polls'
urlpatterns = [
    path('', views.IndexView.as_view(), name='index'),
    path('<int:pk>/', views.DetailView.as_view(), name='detail'),
    path('<int:pk>/results/', views.ResultsView.as_view(), name='results'),
    path('<int:question_id>/vote/', views.vote, name='vote'),
]
```

    <question_id>修改成了<pk>

### 修改視圖

// polls/views.py

刪掉index、detail和results視圖，替換成Django的通用視圖

```py
from django.http import HttpResponseRedirect
from django.shortcuts import get_object_or_404, render
from django.urls import reverse
from django.views import generic

from .models import Choice, Question


class IndexView(generic.ListView):
    template_name = 'polls/index.html'
    context_object_name = 'latest_question_list'

    def get_queryset(self):
        """Return the last five published questions."""
        return Question.objects.order_by('-pub_date')[:5]


class DetailView(generic.DetailView):
    model = Question
    template_name = 'polls/detail.html'


class ResultsView(generic.DetailView):
    model = Question
    template_name = 'polls/results.html'

......

```

ListView和DetailView（它們是作為父類被繼承的）。這兩者分別代表“顯示一個對象的列表”和“顯示特定類型對象的詳細頁面”的抽象概念

`template_name` : 指定模板

`context_object_name` : 指定內容名稱

# tutorial05 測試

## 遇見BUG

在Question.was_published_recently()方法的返回值中，當Qeustion在最近的一天發布的時候返回True（這是正確的），然而當Question在未來的日期內發布的時候也返回True（這是錯誤的）

    python manage.py shell

```py
import datetime
from django.utils import timezone
from polls.models import Question
# 創建一個發布日期在30天后的問卷
future_question = Question(pub_date=timezone.now() + datetime.timedelta(days=30))
# 測試一下返回值
future_question.was_published_recently()
#True
```

## 創建一個測試來暴露這個bug

// polls/tests.py

```py
from django.test import TestCase

# Create your tests here.
import datetime
from django.utils import timezone
from .models import Question

class QuestionMethodTests(TestCase):
    def test_was_published_recently_with_future_question(self):
        """
        在將來發布的問卷應該返回False
        """
        time = timezone.now() + datetime.timedelta(days=30)
        future_question = Question(pub_date=time)
        self.assertIs(future_question.was_published_recently(), False)
```
    
    # 查找投票應用中所有的測試程序
    # 查找名字以test開頭的測試方法
    python manage.py test polls
    
assertIs()方法，它發現was_published_recently()返回了True，而不是我們希望的False

## 修復bug

// polls/models.py

```py
def was_published_recently(self):
    now = timezone.now()
    return now - datetime.timedelta(days=1) <= self.pub_date <= now
```

## 更加全面的測試

```py
class ......

    def test_was_published_recently_with_old_question(self):
        """
        只要是超過1天的問卷，返回 False
        """
        time = timezone.now() - datetime.timedelta(days=1, seconds=1)
        old_question = Question(pub_date=time)
        self.assertIs(old_question.was_published_recently(), False)

    def test_was_published_recently_with_recent_question(self):
        """
        最近一天內的問卷，返回 True
        """
        time = timezone.now() - datetime.timedelta(hours=23, minutes=59, seconds=59)
        recent_question = Question(pub_date=time)
        self.assertIs(recent_question.was_published_recently(), True)
```

# tutorial06 靜態文件

首先在你的polls目錄中創建一個static目錄。 Django將在那裡查找靜態文件

```py
STATIC_URL = '/static/'

# 除了 /static/ 在應用程序中使用目錄外，
# 您還可以STATICFILES_DIRS在設置文件中定義目錄列表（）

""" STATICFILES_DIRS = [
    os.path.join(BASE_DIR, "static"),
] """
```

良好的目錄結構是每個應用都應該創建自己的urls、forms、views、models、templates和static，每個templates包含一個與應用同名的子目錄，每個static也包含一個與應用同名的子目錄

// polls/static/polls/style.css

```css
li a {
    color: green;
}
```

// polls/templates/polls/index.html

```html
{% load static %}

<link rel="stylesheet" type="text/css" href="{% static 'polls/style.css' %}">
```

# tutorial07 自定義admin

## 自定義後台表單

修改admin表單默認排序

// polls/admin.py

```py
from django.contrib import admin
from .models import Question


class QuestionAdmin(admin.ModelAdmin):
    fields = ['pub_date', 'question_text']

admin.site.register(Question, QuestionAdmin)
```

上面的修改讓Publication date字段顯示在Question字段前面了（默認是在後面）

當表單含有大量字段的時候，你也許想將表單劃分為一些字段的集合。再次修改

// polls/admin.py

```py
from django.contrib import admin

from .models import Question


class QuestionAdmin(admin.ModelAdmin):
    fieldsets = [
        (None,               {'fields': ['question_text']}),
        ('Date information', {'fields': ['pub_date']}),
    ]

admin.site.register(Question, QuestionAdmin)
```

## 添加關聯對象

但是一個Question有多個Choices，如果想顯示Choices的內容怎麼辦？有兩個辦法可以解決這個問題

第一個是像Question一樣將Choice註冊到admin站點

// polls/admin.py

```py
from django.contrib import admin
from .models import Choice, Question

# ...
admin.site.register(Choice)
```

Django在admin站點中，自動地將所有的外鍵關係展示為一個select框

如果在創建Question對象的時候就可以直接添加一些Choice，那會更好，這就是我們要說的第二種方法。

// polls/admin.py 修改成以下

```py
from django.contrib import admin
from .models import Choice, Question

class ChoiceInline(admin.StackedInline):
    model = Choice
    extra = 3

class QuestionAdmin(admin.ModelAdmin):
    fieldsets = [
        (None,               {'fields': ['question_text']}),
        ('Date information', {'fields': ['pub_date'], 'classes': ['collapse']}),
    ]
    inlines = [ChoiceInline]

admin.site.register(Question, QuestionAdmin)
```

上面的代碼相當於告訴Django，Choice對象將在Question管理頁面進行編輯，默認情況，請提供3個Choice對象的編輯區域

上面頁面中插槽縱隊排列的方式需要佔據大塊的頁面空間，查看起來很不方便。為此，Django提供了一種扁平化的顯示方式，你僅僅只需要修改一下ChoiceInline繼承的類為admin.TabularInline

```py
# polls/admin.py
class ChoiceInline(admin.TabularInline):
    #...
```

## 定制實例的列表頁面

### list_display

通常，Django只顯示__str()__方法指定的內容。但是很多時候，我們可能要同時顯示一些別的內容。要實現這一目的，可以使用list_display屬性

```py
# polls/admin.py
class QuestionAdmin(admin.ModelAdmin):
    # ...
    list_display = ('question_text', 'pub_date', 'was_published_recently')
```

額外的，我們把was_published_recently()方法的結果也顯示出來

可以通過給方法提供一些屬性來改進輸出的樣式，如下面所示。注意這次修改的是polls/models.py文件

```py
# polls/models.py

class Question(models.Model):
    # ...
    def was_published_recently(self):
        now = timezone.now()
        return now - datetime.timedelta(days=1) <= self.pub_date <= now
    was_published_recently.admin_order_field = 'pub_date'
    was_published_recently.boolean = True
    was_published_recently.short_description = 'Published recently?'
```

### list_filter

還可以對顯示結果進行過濾!使用list_filter屬性，在polls/admin.py

```py
# polls/models.py

class QuestionAdmin(admin.ModelAdmin):
    # ...
    list_filter = ['pub_date']
```

### search_fields

添加一些搜索的能力

```py
# polls/models.py

class QuestionAdmin(admin.ModelAdmin):
    # ...
    search_fields = ['question_text']
```

## 定制admin整體界面

在manage.py文件同級下創建一個templates目錄。然後，打開設置文件mysite/settings.py，在TEMPLATES條目中添加一個DIRS選項：

```py
# mysite/settings.py
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')],  # 要有這一行，如果已經存在請保持原樣
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

### 模板的組織方式

強烈建議每一個模板都應該存放在它所屬應用的模板目錄內（例如polls/templates）而不是整個項目的模板目錄（templates），因為這樣每個應用才可以被方便和正確的重用。只有對整個項目有作用的模板文件才放在根目錄的templates中，比如admin界面

### 修改標題

// mysite\templates\admin\base_site.html

```html
{% extends "admin/base.html" %}

{% block title %}{{ title }} | {{ site_title|default:_('Django site admin') }}{% endblock %}

{% block branding %}
<h1 id="site-name"><a href="{% url 'admin:index' %}">投票站點管理界面</a></h1>
{% endblock %}

{% block nav-global %}{% endblock %}
```

## 定制admin首頁

Django的源代碼

    python -c "import django; print(django.__path__)"

// django\contrib\admin\templates\admin
