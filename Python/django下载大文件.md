基于Django建立的网站，如果提供文件下载功能，最简单的方式莫过于将静态文件交给Nginx等处理，但有些时候，由于网站本身逻辑，需要通过Django提供下载功能，如页面数据导出功能（下载动态生成的文件）、先检查用户权限再下载文件等。因此，有必要研究一下文件下载功能在Django中的实现。



## 最简单的文件下载

**使用HttpResponse**：将文件流放入HttpResponse对象即可，如：

```Python
def file_download(request):
  # do something...
  with open('file_name.txt') as f:
    c = f.read()
  return HttpResponse(c)
```

这种方式简单粗暴，适合小文件的下载，但如果这个文件非常大，这种方式会占用大量的内存，甚至导致服务器崩溃



## 合理的文件下载

**使用StreaminghttpResponse**：Django的HttpResponse对象允许将迭代器作为传入参数，将上面代码中的传入参数c换成一个迭代器，便可以将上述下载功能优化为对大小文件均适合；而Django更进一步，推荐使用 StreamingHttpResponse对象取代HttpResponse对象，StreamingHttpResponse对象用于将文件流发送给浏览器，与HttpResponse对象非常相似，对于文件下载功能，使用StreamingHttpResponse对象更合理

应该先写一个迭代器，用于处理文件，然后将这个迭代器作为参数传递给StreaminghttpResponse对象

```python
from django.http import StreamingHttpResponse

def big_file_download(request):
  # do something...

  def file_iterator(file_name, chunk_size=512):
    with open(file_name) as f:
      while True:
        c = f.read(chunk_size)
        if c:
          yield c
        else:
          break

  the_file_name = "file_name.txt"
  response = StreamingHttpResponse(file_iterator(the_file_name))

  return response
```



## 配置响应头参数

需要让浏览器写入硬盘，而不是展示在浏览器，需要配置响应头参数：

```python
# 根据需要导出文件的类型和文件名称来配置请求头
filename = parse.quote(f"{file_name}.xls".encode("utf-8"))
response = StreamingHttpResponse(file_iterator(c))
# response['content_type'] = 'application/vnd.ms-excel'
response['Content-Type'] = 'application/octet-stream'
# response['Access-Control-Expose-Headers'] = "Content-Disposition" #可不要
response['Content-Disposition'] = 'attachment;filename="{0}"'.format(filename)
```



## 简单的代码示例：

**示例一：下载服务器保存的文件**

```python
from django.http import StreamingHttpResponse

def big_file_download(request):
  # do something...

  def file_iterator(file_name, chunk_size=512):
    with open(file_name) as f:
      while True:
        c = f.read(chunk_size)
        if c:
          yield c
        else:
          break

  the_file_name = "big_file.pdf"
  response = StreamingHttpResponse(file_iterator(the_file_name))
  response['Content-Type'] = 'application/octet-stream'
  response['Content-Disposition'] = 'attachment;filename="{0}"'.format(the_file_name)

  return response
```

**示例二：页面数据导出功能（下载动态生成的文件）**

完整示例，导出到excel

```python
from django.http import StreamingHttpResponse
xls_max_sheet_name_length = 31

def file_iterator(output):
    for x in output:
        yield x

def export_by_queryset(data, file_name, headers=None, column_list=None):
    """
    功能描述：导出到excel;
    请求参数：data：数据-->eg：Queryset对象，或者如 [{},{}] 形式
            file_name:文件名，header:表头, column_list:字段列表，用于指定导出的列名
    返回：写入文件流的响应对象
    """
    wb = xlwt.Workbook(encoding='utf-8')
    sheet = wb.add_sheet(file_name[:xls_max_sheet_name_length])
    row = 0

    if headers:
        for i in range(len(headers)):
            sheet.write(0, i, str(headers[i]))
        row = 1

    if isinstance(data, query.QuerySet):
        data = [model_to_dict(obj) for obj in data]

    if data:
        column_names =column_list if column_list else data[0].keys()
        for d in data:
            col = 0
            for col_name in column_names:
                if isinstance(d.get(col_name),datetime.datetime):
                    value = d.get(col_name).strftime('%Y-%m-%d %X')
                elif isinstance(d.get(col_name),datetime.date):
                    value = d.get(col_name).strftime('%Y-%m-%d')
                else:
                    value = d.get(col_name)
                sheet.write(row, col, value)
                col += 1
            row += 1

    output = io.BytesIO()
    wb.save(output)
    output.seek(0)
        
    filename = parse.quote(f"{file_name}.xls".encode("utf-8"))
    response = StreamingHttpResponse(file_iterator(output))
    # response = HttpResponse(output.getvalue())
    # response['content_type'] = 'application/vnd.ms-excel'
    response['Content-Type'] = 'application/octet-stream'
    response['Access-Control-Expose-Headers'] = "Content-Disposition"
    response['Content-Disposition'] = 'attachment;filename="{0}"'.format(filename)
    return response


class TestView(viewsets.ModelViewSet):
    ......
    @action(methods=['GET'], detail=False)
    def download(request):
        # do something...
        queryset = self.filter_queryset(self.get_queryset())
        response = self.export(queryset, self.file_name, headers=self.columns_cn, column_list=self.columns_en)
        return response
```





