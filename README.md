# Django REST framework Django REST Swagger

[TOC]

# 资源

http://www.django-rest-framework.org/tutorial/quickstart/

http://www.itcto.com.cn/post/2016/10/29/django-rest-framework-restful-api-16102975186.aspx

https://darkcooking.gitbooks.io/django-rest-framework-cn/content/chapter6.html


github：


https://github.com/marcgibbons/django-rest-swagger

# 目标
我想要做什么？
了解rest fremework。
了解django如何生成rest

这里有两个部分。
一个是用djangorestframework 生成rest
另一个是Django REST Swagger 用来生成schma界面的。



# 项目进度

# 遇到的问题

# 总结



# 第一章 Tutorial 1: Serialization 序列化

## 1. 安装必要组件

pip install django
pip install djangorestframework
pip install pygments  # We'll be using this for the code highlighting

## 2. 新建app

python manage.py startapp snippets

## 3. settings里添加snippets和rest_framework

INSTALLED_APPS里添加就行了。

## 4. 新建model

```python
from django.db import models
from pygments.lexers import get_all_lexers
from pygments.styles import get_all_styles

LEXERS = [item for item in get_all_lexers() if item[1]]
LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
STYLE_CHOICES = sorted((item, item) for item in get_all_styles())


class Snippet(models.Model):
    created = models.DateTimeField(auto_now_add=True)
    title = models.CharField(max_length=100, blank=True, default='')
    code = models.TextField()
    linenos = models.BooleanField(default=False)
    language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
    style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

    class Meta:
        ordering = ('created',)
```

sort有点没看懂
python manage.py makemigrations snippets
python manage.py migrate


## 5. 新建一个序列化类

第一步我们需要为API提供一个序列化以及反序列化的方法，用来把snippet对象转化成json数据格式，创建serializers和创建Django表单类似。我们在snippets目录创建一个serializers.py：

```python
from rest_framework import serializers
from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


class SnippetSerializer(serializers.Serializer):
    pk = serializers.IntegerField(read_only=True)
    title = serializers.CharField(required=False, allow_blank=True, max_length=100)
    code = serializers.CharField(style={'base_template': 'textarea.html'})
    linenos = serializers.BooleanField(required=False)
    language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
    style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

    def create(self, validated_data):
        """
        如果数据合法就创建并返回一个snippet实例
        """
        return Snippet.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """
        如果数据合法就更新并返回一个存在的snippet实例
        """
        instance.title = validated_data.get('title', instance.title)
        instance.code = validated_data.get('code', instance.code)
        instance.linenos = validated_data.get('linenos', instance.linenos)
        instance.language = validated_data.get('language', instance.language)
        instance.style = validated_data.get('style', instance.style)
        instance.save()
        return instance
```


在第一部分我们定义了序列化/反序列化的字段，creat()和update()方法则定义了当我们调用serializer.save()时如何来创建或修改一个实例。

serializer类和Django的Form类很像，并且包含了相似的属性标识，比如required、 max_length和default。

这些标识也能控制序列化后的字段如何展示，比如渲染成HTML。上面的{'base_template': 'textarea.html'}和在Django的Form中设定widget=widgets.Textarea是等效的。 这一点在控制API在浏览器中如何展示是非常有用的，我们后面的教程中会看到。

我们也可以使用ModelSerializer类来节省时间，后续也会介绍，这里我们先显式的定义serializer。


## 6. 使用Serializers(就是shell里实际操作一下)

这里就是使用shell，各种操作。

## 7. 使用ModelSerializers

snippet模型和SnippetSerializer有太多重复代码。

就像django有form和ModelForm类。
rest framework也有 Serializer类和ModelSerializer。

```python
class SnippetSerializer(serializers.ModelSerializer):
    class Meta:
        model = Snippet
        fields = ('id', 'title', 'code', 'linenos', 'language', 'style')
```
可以通过打印输出来检查serializer包含哪些字段，打开Django shell并输入一下代码：

```python
from snippets.serializers import SnippetSerializer
serializer = SnippetSerializer()
print(repr(serializer))
```

## 8. 使用Serializer编写常规的Django视图

首先创建了一个返回json格式数据的方法。
返回的是httpresponse的子类。
也就是http申请过去返回来的数据。

然后主页函数，
如果是get那么展示list，如果是post那么新建一个。

还需要一个页面用来展示，修改或者删除某个具体的snippet

```python
from django.shortcuts import render
from django.http import HttpResponse
from django.views.decorators.csrf import csrf_exempt
from rest_framework.renderers import JSONRenderer
from rest_framework.parsers import JSONParser
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer
# Create your views here.


class JSONResponse(HttpResponse):
    """
    用于返回JSON数据.
    """
    def __init__(self, data, **kwargs):
        content = JSONRenderer().render(data)
        kwargs['content_type'] = 'application/json'
        super(JSONResponse, self).__init__(content, **kwargs)



@csrf_exempt
def snippet_list(request):
    """
    展示所以snippets,或创建新的snippet.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return JSONResponse(serializer.data)

    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(data=data)
        if serializer.is_valid():
            serializer.save()
            return JSONResponse(serializer.data, status=201)
        return JSONResponse(serializer.errors, status=400)


@csrf_exempt
def snippet_detail(request, pk):
    """
    修改或删除一个snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return HttpResponse(status=404)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return JSONResponse(serializer.data)

    elif request.method == 'PUT':
        data = JSONParser().parse(request)
        serializer = SnippetSerializer(snippet, data=data)
        if serializer.is_valid():
            serializer.save()
            return JSONResponse(serializer.data)
        return JSONResponse(serializer.errors, status=400)

    elif request.method == 'DELETE':
        snippet.delete()
        return HttpResponse(status=204)
```

## 9. 修改url

```python
    url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
    url(r'^snippets/$', views.snippet_list),
```

runserver之后 http://127.0.0.1:8000/snippets/
那么就得到了lists

# 第一章-旁支-Django Rest Swagger

## 1. 安装和设置

 pip install django-rest-swagger

 settings.py INSTALLED_APPS  'rest_framework_swagger',

## 2. url

主要在这里设置。
但是我发现，如果没有设置viewset，也就是没有完成rest，这个设置是没有用处的。

# 第二章 Requests和Responses

## 1. Request对象

REST framework使用一个叫Requests的对象扩展了原生的HttpRequest，并提供了更灵活的请求处理。Requests对象的核心属性就是request.data，和requests.POST类似，但更强大：
request.POST  # 只处理form数据.只接受'POST'方法.
request.data  # 处理任意数据.接受'POST','PUT'和'PATCH'方法.

## 2. Response对象

REST framework也提供了一个类型为TemplateResponse的Response对象，它返回类型由数据决定。(原文如下，这句话我怎么翻译都觉得别扭，意思就是客户端要神码类型它返回神码类型)
return Response(data)  # 返回类型由发出请求的客户端决定

## 3. 状态码

使用数字HTTP状态码并不总是易于阅读的，并且当你得到一个错误的状态码时不容易引起注意。REST framework为每个状态码都提供了更明显的标志，比如status模块中的HTTP_400_BAD_REQUEST，使用它们替代数字是一个好注意。

## 4. 装饰API视图

REST framework提供了2种装饰器来编写视图：

    基于函数视图的@api_view
    基于类视图的APIView

这些装饰器提供了少许功能，比如确保在视图中接收Request实例，添加context到Resonse对象来决定返回类型。

此外还提供了适当的错误处理，比如405 Method Not Allowed、有畸形数据传入时引发的ParseError异常等。

## 5. 协同工作

现在我们使用这些组件来改写views.

我们不再需要JSONResponse类了，删除它后修改代码如下：
用上面这些response什么的，就不需要json修改函数了。
主要区别就是，@api_view导入get和post。
然后response原来是jsonparse函数。
状态码也改了。
有个问题，put和post有啥区别？
貌似没太大区别，就是格式不一样。



```python
from rest_framework import status
from rest_framework.decorators import api_view
from rest_framework.response import Response
from snippets.models import Snippet
from snippets.serializers import SnippetSerializer


@api_view(['GET', 'POST'])
def snippet_list(request):
    """
    展示或创建snippets.
    """
    if request.method == 'GET':
        snippets = Snippet.objects.all()
        serializer = SnippetSerializer(snippets, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        serializer = SnippetSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

改造后的代码更简洁了，并且更像Forms。也使用了命名后的状态码让Response含义更加明显。

接下来修改单一的snippets视图：

@api_view(['GET', 'PUT', 'DELETE'])
def snippet_detail(request, pk):
    """
    修改或删除一个snippet.
    """
    try:
        snippet = Snippet.objects.get(pk=pk)
    except Snippet.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = SnippetSerializer(snippet)
        return Response(serializer.data)

    elif request.method == 'PUT':
        serializer = SnippetSerializer(snippet, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    elif request.method == 'DELETE':
        snippet.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)

```

一切都是如此熟悉，这和原生的Django Form很像。

注意我们并没有显示的声明request和response中的内

## 6. 为URL添加可选的数据格式后缀

我们的responses支持多种返回格式，利用这点我们可以通过在URL中添加格式后缀的方法来获取单一数据类型，这意味着我们的URL可以处理类似http://example.com/api/items/4/.json这样的格式。

首先在views添加format参数：

def snippet_list(request, format=None):

def snippet_detail(request, pk, format=None):

然后修改urls.py，添加format_suffix_patterns：

```python
from django.conf.urls import url
from rest_framework.urlpatterns import format_suffix_patterns
from snippets import views

urlpatterns = [
    url(r'^snippets/$', views.snippet_list),
    url(r'^snippets/(?P<pk>[0-9]+)$', views.snippet_detail),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

我们不需要添加任何额外信息，它给我们一个简单清晰的方法去获取指定格式的数据。
http://127.0.0.1:8000/snippets/?format=api


这个时候，看到swagger居然能用了。

http://127.0.0.1:8000/

http://127.0.0.1:8000/#/snippets

## 7. 测试

```python
和上一章一样，我们使用命令行来进行测试。尽管添加了相应的错误处理，但一切还是那么简单：

http http://127.0.0.1:8000/snippets/

HTTP/1.1 200 OK
...
[
  {
    "id": 1,
    "title": "",
    "code": "foo = \"bar\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  },
  {
    "id": 2,
    "title": "",
    "code": "print \"hello, world\"\n",
    "linenos": false,
    "language": "python",
    "style": "friendly"
  }
]

我们可以通过控制Accept来控制返回的数据类型：

http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON
http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML

或者通过添加格式后缀：

http http://127.0.0.1:8000/snippets.json  # JSON suffix
http http://127.0.0.1:8000/snippets.api   # Browsable API suffix

同样的，我们可以通过Content-Type控制发送请求的数据类型：

# POST using form data
http --form POST http://127.0.0.1:8000/snippets/ code="print 123"

{
  "id": 3,
  "title": "",
  "code": "print 123",
  "linenos": false,
  "language": "python",
  "style": "friendly"
}

# POST using JSON
http --json POST http://127.0.0.1:8000/snippets/ code="print 456"

{
    "id": 4,
    "title": "",
    "code": "print 456",
    "linenos": false,
    "language": "python",
    "style": "friendly"
}

或打开浏览器访问http://127.0.0.1:8000/snippets/
```

# 第三章 类视图

我们也可以使用类视图来编写API的view,而不是函数视图。正如我们了解的，类视图是一种非常给力的模式让我们重用代码，

## 1. 使用类视图重写API

