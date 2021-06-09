# 文件上传

**后端：**

```python
def check_upload_file(file, filename):
    """上传文件校验"""
    size = len(file.read())
    if size > 1024 * 1024 * 20:
        raise ParamException(msg='文件大小超出限制，仅限20MB以下!')

    files = os.listdir(REFER_TO_FILE_DIR)
    if filename in files:
        raise ParamException(msg='目录下已存在同名文件!')
        
class ReferToFileList(Resource):
    """参考文件展示，上传"""
    def post(self):
        file = request.files.get('file')
        filename = file.filename
        # filename = secure_filename(file.filename)
        check_upload_file(file, filename)
        filepath = os.path.join(REFER_TO_FILE_DIR, filename)
        try:
            file.save(filepath)
        except Exception as e:
            # logger.error(e)
            raise ParamException(msg='上传失败!')
        return render(msg="上传成功!", filename=filename)
```

**前端vue：**

html:  手动ajax请求上传

```html
<el-upload class="upload-demo" ref="upload" action :before-upload="beforeUploadFile" :on-preview="handlePreview" :on-exceed="overLimit"
          :on-change="handleChange" :on-remove="handleRemove" :limit="1" :file-list="upload.fileList" :auto-upload="false">
          <el-button size="medium" slot="trigger">选取文件</el-button>
          <el-button size="medium" @click="save" type="primary" icon="el-icon-upload" :loading="upload.loading">上传</el-button>
        </el-upload>

```

js：

```js
	  handlePreview() {},
          
      /* 添加文件，文件上传成功或失败时的回调 */
      handleChange(file, fileList) {
        // 添加文件时就做文件限制大小提示，但没法限制用户上传（由于before-upload手动上传无法触发）
        const isLimitSize = file.size / 1024 / 1024 < 20 // 大小不超过20M
        if (!isLimitSize){
          this.$message.warning('上文件大小不能超过20MB,当前文件已超限制!')
        }
      },
      handleRemove() {},
      exportTemplate() {},
          
      /* 文件个数超出限制时的行为  */
      overLimit(files, fileList) {
        // 1.不覆盖已选文件：只提示超限制，
        this.$message.warning('一次只能上传 1 个文件, 重新选择请先移除列表中的文件!');
        // 该files中的元素file和其他回调中的file(或fileList中的元素file)格式不一样，只是其中(后者)的一个raw属性,也就是File文件对象
        // 2.直接覆盖原件：limit限制为1时，添加文件若超出数量，直接覆盖
        // this.$set(fileList[0], 'raw', files[0])
        // this.$set(fileList[0], 'name', files[0].name)
        // this.$set(fileList[0], 'size', files[0].size)
        // this.$set(fileList[0], 'status', files[0].status)
      },
          
      /* 文件上传前的行为，若设置aoto-upload=false，上传文件的回调都不会触发 */
      beforeUploadFile(file){
        const isLimitSize = file.size / 1024 / 1024 < 20 // 大小不超过20M
        if (!isLimitSize){
          this.$utils.errorMessage('上传文件大小不能超过20MB!')
        }
        return isLimitSize
      },

      // 文件上传
      save() {
        this.upload.loading = true
        var formData = new FormData()
        if (this.$refs.upload.uploadFiles.length > 0) {
          formData.append('file', this.$refs.upload.uploadFiles[0].raw)
        }
        this.$httpd.post('/ToMaintain/ReferToFileList', formData)
          .then(res => {
            this.$utils.successMessage(res.data.msg)
            this.files.push(res.data.filename)
            this.upload.loading = false
          }).catch(err => {
            this.upload.loading = false
            // this.$utils.errorMessage(err.data.message)
          })
      },
```





# 文件下载

文件数据小的话，可以直接使用flask自带的`send_file()` 读取文件或者直接写入缓冲区，再去读取都可以，`StringIO`, `ByteIO`,这样就是能写成不生层本地文件去下载的模式了

### 网页GET请求接口下载文件

网址输入后端API地址访问下载；或者window.location.href ，或者a标签点击下载；接口将服务器上指定路径下的文件使用flask自带的`send_file`方法返回.

此方法需要指定头 'content-disposition' ："attachment;" ,表示以附件形式处理

```python
from flask import request, make_response, send_file
from urllib.parse import quote

@api.route(methods=["GET"], rule="/ToMaintain/ReferTofile")
def download():
    filename = request.args.get('filename') # 文件名，带后缀
    filepath = os.path.join(UPLOAD_DIR, filename) # 文件绝对路径
    response = make_response(send_file(filepath))
    # 文件名需要特别处理，否则报错 'latin-1' encode错误
    response.headers['content-disposition'] = "attachment; filename={}".format(filename.encode().decode('latin-1'))
    
    # 使用send_file时：content-type默认为 application/octet-stream
    # response.headers['content-type'] = 'application/octet-stream'
    
    # 也可以直接在请求头传入文件名，方便前端获取；处理中文名：后端url转码，前端使用decodeURI()解码
    # response.headers['filename'] = quote(filename.encode('utf-8'))
    return response
```

##### 流式下载

```python
def file_iterator(filepath, chunk_size=512):
    """文件读取迭代器
    file_path：路径
    chunk_size：每次读取大小
    """
    with open(filepath, 'rb') as f:
        while 1:
            chunk = f.read(chunk_size)
            if not chunk:
                break
            yield chunk

@api.route(methods=["GET"], rule="/ToMaintain/ReferTofile")
def download():
    filename = request.args.get('filename') # 文件名，带后缀
    filepath = os.path.join(UPLOAD_DIR, filename) # 文件绝对路径
    # response = Response(stream_with_context(logic.file_iterator(filepath)))
    response = Response(file_iterator(filepath), content_type="application/octet-stream")
    response.headers['filename'] = quote(filename.encode('utf-8'))
    response.headers['Access-Control-Expose-Headers'] = 'filename'
    return response
```

```
运行 Flask app，可以正确下载文件，但是下载只有实时速度，没有文件总大小，导致无法知道下载进度，也没有文件类型，这些我们都可以通过增加 header 字段实现：
response = Response(generate(), mimetype='application/gzip')
response.headers['Content-Disposition'] = 'attachment; filename={}.tar.gz'.format("download_file")
response.headers['content-length'] = os.stat(str(file_path)).st_size
return response
```



### ajax请求实现文件下载

**后端：flask后端直接使用`send_file`返回文件流即可：**

```python
class ReferTofile(Resource):
    """文件下载"""
    def post(self):
        """下载"""
        filename = request.json.get('filename')
        filepath = os.path.join(UPLOAD_DIR, filename)
        return send_file(filepath)

    
# 也可以使用make_response添加请求头参数，比如添加文件名
response = make_response(send_file(filepath))
response.headers['filename'] = quote(filename.encode('utf-8')) # 前端使用decodeURI()解码获取
# 前端想要访问到其他的响应头的话 需要在服务器上设置 Access-Control-Expose-Headers
response.headers['Access-Control-Expose-Headers'] = 'filename'
        
        
```

默认请求，前端只能访问以下默认响应头信息：

- Cache-Control
- Content-Language
- Content-Type
- Expires
- Last-Modified
- Pragma

若想要访问其他响应头信息或自定义响应头，需要在服务器上设置 `Access-Control-Expose-Headers`

```
Access-Control-Expose-Headers : 'filename'
```



**前端： 请求回文件流后，使用`blob`对象，将文件流写入文件：**

```js
// 处理请求返回的文件流；
// 创建blob对象，写入文件流；创建a标签点击url，触发浏览器下载
hanldeFileStream(response) {
    filename = decodeURI(response.headers.get('filename')) //获取请求头中的文件名
    let blob = new Blob([response.data])
    if (window.navigator.msSaveOrOpenBlob) {
        navigator.msSaveBlob(blob)
    } else {
        let link = document.createElement('a')
        link.download = filename
        link.style.display = 'none'
        link.href = URL.createObjectURL(blob) // 将blob放入url下
        document.body.appendChild(link)
        link.click()
        URL.revokeObjectURL(link.href)
        document.body.removeChild(link)
    }
},

// 1.post请求后端接口，返回文件流
download(item){
    this.$httpd.post(
        url, 
        {filename:item}, 
        {
            responseType: 'blob', // 必须指定响应体类型为blob，否则乱码显示在网页上
            // timeout: 1000 * 60 * 60 * 24
        },
    ).then(
        response => {
            this.hanldeFileStream(response,item)
            this.$message.success('导出成功')
            this.upload.loading = false
        }
    ).catch(error => {
        console.log(error)
        console.log(error.response)
        this.upload.loading = false
        this.$message.error("导出文件失败")
    })   
}

// 2.get 请求后端接口，返回文件流
download(filename){
    this.$httpd.get('/ToMaintain/Opinion', {
        params:{
            filename:filename,
            board_code:this.board_code,
        },
        responseType: 'blob',
    }).then(res=>{
        this.$utils.hanldeStream(res)
    })
},
```

- 指定 responseType: 'blob', 将响应体以Blob对象返回，形如：{data:Blob, status:200, statusText:"ok",......}







