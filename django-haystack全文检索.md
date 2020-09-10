

# 使用haystack实现django全文检索搜索引擎功能

```
haystack支持多种搜索引擎，不仅仅是whoosh，使用solr、elastic search等搜索，也可通过haystack，而且直接切换引擎即可，甚至无需修改搜索代码。

中文搜索需要进行中文分词，使用**jieba**。
```

## 一，基本使用（前后不分离）

### 1.安装相关包

```
pip install django-haystack
pip install whoosh
pip install jieba
```



### 2.配置django的settings

添加app

```
NSTALLED_APPS = (
    ...
    'haystack',
)
```

haystack相关默认配置

```python
# haystack的相关配置：
HAYSTACK_CONNECTIONS = {
    'default': {
    	# 配置引擎，这里使用whoosh
    	# whoosh引擎默认英文分词
        # 'ENGINE': 'haystack.backends.whoosh_backend.WhooshEngine',
        # 配置whoosh引擎为jieba中文分词
        'ENGINE': 'snippets.whoosh_cn_backend.WhooshEngine',  # 使用中文jieba分词
        'PATH': os.path.join(BASE_DIR, 'whoosh_index'),
    }
}
# 每页显示搜索结果数目为10
HAYSTACK_SEARCH_RESULTS_PER_PAGE = 10

# 添加此项，当添加、修改、删除数据时，会自动更新索引
HAYSTACK_SIGNAL_PROCESSOR = 'haystack.signals.RealtimeSignalProcessor'
```



### 3.添加url

在整个项目的urls.py中，配置搜索功能的url路径：

```python
urlpatterns = [
    ...
    url(r'^search/', include('haystack.urls')),# 该url自带一个默认类视图
]
```

haystack.urls的内容为:

```python
from django.conf.urls import url
from haystack.views import SearchView

urlpatterns = [
    url(r'^$', SearchView(), name='haystack_search'),
]
```

### 4.添加一个使用索引文件

在子应用app目录下，创建`search_indexes.py`文件

```python
from haystack import indexes

from snippets.models import Snippet
# 类名为模型名+Index
class SnippetIndex(indexes.SearchIndex, indexes.Indexable):
    text = indexes.CharField(document=True, use_template=True, 				  template_name='search/indexes/snippets/snippet_text.txt')
    # 其他的字段只是附属的属性，方便调用，并不做检索的依据。
    # title = indexes.CharField(model_attr='title')
    # code = indexes.CharField(model_attr='code')

    def get_model(self):
        return Snippet

    def index_queryset(self, using=None):
        """Used when the entire index for model is updated."""
        return self.get_model().objects.all()
```



### 5.指定索引模板文件

在项目的根目录下添加（最好在settings中配置templates）：“templates/search/indexes/应用名称/”下创建“模型类名称小写_text.txt”文件。例：--->snippet_text.txt

此文件指定将模型中的哪些字段建立索引，搜索时在该文件指定字段上生成的索引去查找

```
{{object.title}}
{{object.字段2}}
{{object.字段3}}
```

**注意：如果更改了以上文件索引字段选择，则需要重新运行以下命令生成索引：**

```
python manage.py rebuild_index
```



### 6.指定搜索结果页面

在templates/search/下面，建立一个search.html页面

```html
<!DOCTYPE html>
<html>
<head>
    <title></title>
</head>
<body>
{% if query %}
    <h3>搜索结果如下：</h3>
    {% for result in page.object_list %}

        <a href="/Snippets/snippet/{{ result.object.id }}/">{{ result.object.title }}</a><br/>
    {% empty %}
        <p>啥也没找到</p>
    {% endfor %}

    {% if page.has_previous or page.has_next %}
        <div>
            {% if page.has_previous %}<a href="?q={{ query }}&amp;page={{ page.previous_page_number }}">{% endif %}&laquo; 上一页{% if page.has_previous %}</a>{% endif %}
        |
            {% if page.has_next %}<a href="?q={{ query }}&amp;page={{ page.next_page_number }}">{% endif %}下一页 &raquo;{% if page.has_next %}</a>{% endif %}
        </div>
    {% endif %}
{% endif %}
</body>
</html>
```



### 7.使用jieba中文分词器

```
在django-haystack模块安装目录下：路径 D:\venv\myproject\Lib\site-packages\django_haystack-2.8.1-py3.7.egg\haystack\backends 目录下，复制其中 `whoosh_backend.py`更名`whoosh_cn_backend.py`放入该app目录下：
```

`whoosh_backend.py`就是whoosh引擎配置所在文件

**a.修改其中分词器部分：**

```python
# 在整个py文件中，查找
analyzer=StemmingAnalyzer()
# 改为 (应该就一处)
analyzer=ChineseAnalyzer()
```

所需修改代码所在文件位置：

```python
   elif field_class.field_type == 'edge_ngram':
        schema_fields[field_class.index_fieldname] = NGRAMWORDS(minsize=2, maxsize=15, at='start', stored=field_class.stored, field_boost=field_class.boost)
   else:
        # schema_fields[field_class.index_fieldname] = TEXT(stored=True, analyzer=StemmingAnalyzer(), field_boost=field_class.boost, sortable=True)
        # 改为jieba中文分词
        schema_fields[field_class.index_fieldname] = TEXT(stored=True, analyzer=ChineseAnalyzer(), field_boost=field_class.boost, sortable=True)

   if field_class.document is True:
        content_field_name = field_class.index_fieldname
```

**b.可以自定义`analyzer.py`文件放入该app目录下,覆盖jieba原始的分词器文件：**

```python
import jieba
from whoosh.analysis import Tokenizer, Token

class ChineseTokenizer(Tokenizer):
    def __call__(self, value, positions=False, chars=False,
                 keeporiginal=False, removestops=True,
                 start_pos=0, start_char=0, mode='', **kwargs):

        t = Token(positions, chars, removestops=removestops, mode=mode,
                  **kwargs)
        seglist = jieba.cut(value, cut_all=False)  # (精确模式)使用结巴分词库进行分词
        # seglist = jieba.cut_for_search(value)  # (搜索引擎模式) 使用结巴分词库进行分词
        for w in seglist:
            t.original = t.text = w
            t.boost = 1.0
            if positions:
                t.pos = start_pos + value.find(w)
            if chars:
                t.startchar = start_char + value.find(w)
                t.endchar = start_char + value.find(w) + len(w)
            yield t  # 通过生成器返回每个分词的结果token


def ChineseAnalyzer():
    return ChineseTokenizer()
```



### 8.生成索引

手动生成一次索引

```
python manage.py rebuild_index
```

### 9.使用

网页输入以下，会自动使用search.html模板展示搜索结果

```
http://127.0.0.1:8001/search/?q=关键字
```



## 二，自定义

### 自定义搜索view

将搜索的url导入到自定义view中处理

```python
from django.http import JsonResponse
# from django.middleware import csrf
from haystack.views import SearchView

class MysearchView(SearchView):
    # haystack 默认是有几个变量的，这几个变量存在context的，最终前端接受的数据都在context字典里，所以如果我们定义自己的变量的话，也需要加载context里：
    # 通过重写extra_context 来添加自己的变量，
    # 通过看源码，extra_context 默认返回的是空，然后再get_context方法里面，把extra_context
    # 返回的内容加到我们self.context字典里
    def create_response(self):
        """Generates the actual HttpResponse to send back to the user."""
        context = self.get_context()
        keyword = self.request.GET.get('q', None) # 关键字参数为q
        if not keyword:
            return JsonResponse({'msg':'无相关信息'})
        else:
            data_list = []
            for obj in context['page'].object_list:
                data_dict = {}
                data_dict['id'] = obj.object.id
                data_dict['title'] = obj.object.title
                data_dict['code'] = obj.object.code
                data_list.append(data_dict)
            print(context)
            return JsonResponse({'res': data_list})
```

urls.py:

```
urlpatterns = [
	......
    path('search/', MysearchView(),name='haystack_search')
]
```

网页输入：

```
http://127.0.0.1:8001/api-auth/search/?q=傻
```

