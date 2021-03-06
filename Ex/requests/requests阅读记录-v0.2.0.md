requests作者是Kenneth Reitz，他的[github地址](https://github.com/kennethreitz)。  
从0.2.0版本开始阅读。  
commit id为 2427eca。  
-----------------------

### 1.测试用例
    def test_HTTP_200_OK_GET(self):
        r = requests.get('http://google.com')
        self.assertEqual(r.status_code, 200)
这里从直接调用requests的get方法。
作者的设计方法是，在requests目录下的__init__.py文件中import当前目录下的core.py文件，get方法实际存放在core.py文件中。

### 2.requests的get方法

    def get(url, params={}, headers={}, auth=None):
    
        r = Request() 		#实例化一个 Request类，见3。
    
        r.method = 'GET'	# 初始化r的各个属性。
        r.url = url
        r.params = params
        r.headers = headers
        r.auth = _detect_auth(url, auth) 	# 权限认证，如果需要的话。
    
        r.send()		# 调用r的send方法，重要，见4。
    
        return r.response	# 返回的是response，即一个Response()实例。
	
### 3.Request()方法
r = Request()会实例化一个 Request类，因为Request类内容太多，这里截取需要的部分，以后也是这样，如有需要，可查看源码。

    class Request(object):
        _METHODS = ('GET', 'HEAD', 'PUT', 'POST', 'DELETE')

        def __init__(self):
            self.url = None
            self.headers = dict()
            self.method = None
            self.params = {}
            self.data = {}
            self.response = Response()
            self.auth = None
            self.sent = False

        def __setattr__(self, name, value):  # 对method做了限制。当试图对象的属性赋值的时候将会被调用。
                if (name == 'method') and (value):
                    if not value in self._METHODS:
                        raise InvalidMethod()

                object.__setattr__(self, name, value)
			
### 4.r.send()方法

    def send(self, anyway=False):
            self._checks()	# 它封装了url是否非为None的判断函数。

            success = False	 # 预设 success 为 False

            if self.method in ('GET', 'HEAD', 'DELETE'): # 对method方法进行判断。
                if (not self.sent) or anyway:

                    if isinstance(self.params, dict):	# 对params的类型进行判断，是否是dict。
                        params = urllib.urlencode(self.params)	# 对参数进行了整合，细节可以忽略。
                    else:
                        params = self.params

                    req = _Request(("%s?%s" % (self.url, params)), method=self.method)
                    # 实例化了_Request类，见5。这行代码结果示例：parms ={'age':23, 'name':wsp}, url='www.baidu.com'，将会转换为 www.baidu.com?age=23&name=wsp 。

                    if self.headers:
                        req.headers = self.headers

                    opener = self._get_opener()	# 如果需要验证，根据urllib提供的函数，进行请求的验证，如果不需要，则直接返回 urllib.urlopen 。

                    try:
                        resp = opener(req) 	# 一层一层往上查，实际上等于 resp = urllib2.urlopen(urllib.Reuqest())了。即是调用标准库发送请求的真正的方法了。
                        self.response.status_code = resp.code		# 对Response类进行赋值，见6。
                        self.response.headers = resp.info().dict
                        if self.method.lower() == 'get':
                            self.response.content = resp.read()

                        success = True	# 设置success为True
                    except urllib2.HTTPError, why: # 这里，pep8已经不推荐写逗号了，用as
                        self.response.status_code = why.code

            self.sent = True if success else False # 这个用法也很有意思。判断success的值，如果是True，则sent就设为True，否则就是False。
                    #	sent的意义是，是否发送成功。

            return success

### 5._Request类。
    class _Request(urllib2.Request): # 该类继承了urllib2.Request类。
        def __init__(self, url,
                        data=None, headers={}, origin_req_host=None,
                        unverifiable=False, method=None):
            urllib2.Request.__init__( self, url, data, headers, origin_req_host,
                                      unverifiable)
            self.method = method

        def get_method(self):
            if self.method:
                return self.method

            return urllib2.Request.get_method(self)

解析
>req = _Request(("%s?%s" % (self.url, params)), method=self.method)：
_Request传入两个参数，第一个是url，第二个是data。其中，url进行了格式化，最终结果示例：www.baidu.com?age=23&name=wsp。
url生成之后，就是_Request的构造方法了。首先调用了urllib2.Request.__init__方法，忽略。然后是self.method = method。
		
### 6.response
Request中有一个字段，self.response = Response()，它实例化了一个Response对象，如下。

    class Response(object):	# 注意，这里的Response类只继承了object，意味着它的所有字段值都是自设的。也就是说，并不一定要严格按照规定，只要数据对，看着像就行了。
        def __init__(self):
            self.content = None
            self.status_code = None
            self.headers = dict()

        def __repr__(self):
            try:
                repr = '<Response [%s]>' % (self.status_code)
            except:
                repr = '<Response object>'
            return repr

    self.response.status_code = resp.code，这段代码的意思是，对级联属性进行赋值。

### 后记：现在的版本，已经不是基于 urllib。