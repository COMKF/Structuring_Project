requests作者是Kenneth Reitz，他的[github地址](https://github.com/kennethreitz)。  
阅读版本为v0.3.0。  
commit id为 044252ea。
-----------------------

## 一 v0.2.1的重要功能 （commit id为 e09efc49）
### 1.post方法测试。
    def test_POSTBIN_GET_POST_FILES(self):

        bin = requests.post('http://www.postbin.org/')
        self.assertEqual(bin.status_code, 200)

        post = requests.post(bin.url, data={'some': 'data'})
        self.assertEqual(post.status_code, 201)

        post2 = requests.post(bin.url, files={'some': StringIO('data')})
        # StringIO 是一个可以把 String 强制转换成 类似于 FILE 类的东西。这是一个文件测试。self.assertEqual(post2.status_code, 201)
        # 其实现在已经对状态码进行了修改，现在post过去的状态码是200。
### 2.send方法
在这个版本里，post方法的主要代码逻辑在send方法里，通过if判断，进行处理的。这里摘取主要代码。

    elif self.method == 'POST':	# 对方法进行判断。
        if (not self.sent) or anyway:

            if self.files:
                register_openers()	# HTTP传输有关，这里不做深究。
                datagen, headers = multipart_encode(self.files)	# 文件传输的方法，见3.
                req = _Request(self.url, data=datagen, headers=headers, method='POST')

                if self.headers:
                    req.headers.update(self.headers)

            else:
                req = _Request(self.url, method='POST')
                req.headers = self.headers

                # url encode form data if it's a dict
                if isinstance(self.data, dict):
                    req.data = urllib.urlencode(self.data)
                else:
                    req.data = self.data

### 3.multipart_encode(self.files)方法。
当post 一个文件的时候，数据要以MIME 格式分块，客户端生成随机的 boundry来标记一块的开头和结尾，并且需要在 header中加入 Content-Length 来表明文件大小。所以这个函数大致是干这些事情。

    def multipart_encode(params, boundary=None, cb=None):
      if boundary is None:
          boundary = gen_boundary()	# 生成随机字符串作为 boundary
      else:
          boundary = urllib.quote_plus(boundary)

      headers = get_headers(params, boundary) 	# 生成Content-Length等头信息
      params = MultipartParam.from_params(params)

      return multipart_yielder(params, boundary, cb), headers # 返回迭代器和头信息

### 4.构成返回的Response实例时，添加了url字段。
    self.response.url = resp.url

## 二 v0.2.2的改进 （commit id为 ed909eaa）
### 1.返回码
当返回状态码不是200（也就是说，请求出错）时，依然有返回内容。（以前返回的内容都是None，相当于修改了下小BUG）。

	try:
		resp = opener(req)
		self._build_response(resp) # 对之前Response实例的赋值，进行了封装
		success = True
	
	except urllib2.HTTPError as why:
		self._build_response(why)	# 直接把why的值赋值给了Response实例
		success = False
		
### 2.异步的monkeypatch支持。

当一个库的代码是纯 Python 的时候，加上 monkey patch 技法，那么这个库使用的就是 gevent.socket 了，从而不需要任何更改就能够获得 gevent 的同步代码异步执行的“超级牛力”。
段位不够，看不懂。说的应该是异步执行的能力。

	try:
		import eventlet
		eventlet.monkey_patch()
	except ImportError:
		pass
	
	if not 'eventlet' in locals():
		try:
			from gevent import monkey
			monkey.patch_all()
		except ImportError:
			pass
		


### 3.cookie 支持。调用urllib2函数,具体的以后再说吧。

## 三 v0.2.3的改进（Response的小优化）（commit id为 53e785f6）
### 1.改进了类 Response 的表现形式。
    def test_nonzero_evaluation(self):
        r = requests.get('http://google.com/some-404-url')
        self.assertEqual(bool(r), False)	# 主要看bool(r)

        r = requests.get('http://google.com/')
        self.assertEqual(bool(r), True)
	
    Response():
        # 新增方法，这个方法可以对类的实例进行 bool 运算，即 bool(r)=r.__nonzero__()。
        # __nonzero__(self): 为python的内置 magic 方法，当对该类调用bool(object) 时调用该方法。
        def __nonzero__(self):
                return not self.error
			
相关细节：

	try:
		resp = opener(req)
		self._build_response(resp)
		self.response.ok = True	# ok字段原本是False，当发送成功时，更改为True。
	
	except urllib2.HTTPError as why:
		self._build_response(why)
		self.response.error = why	# error字段原本是None，当出现异常时，会更改self.error 变量。
		
这个版本没什么好说的，针对返回的类的细节优化。
我觉得自己很少会对返回的东西做细节的优化，好像它可用，我就接受了。
当真的需要把轮子开放给其他开发者时，那种追求卓越就会让你做的更好。
或许这就是一流代码狗和三流的区别。
		
## 四 v0.2.4的改进 （commit id为 53e785f6）
### 1.自动测试验证。
    class RequestsTestSuite(unittest.TestCase):
      def setUp(self):
          pass

      def tearDown(self):
          pass

      def test_invalid_url(self):
          self.assertRaises(ValueError, requests.get, 'hiwpefhipowhefopw')

从开始到现在，无论哪个版本，都有测试文件，且测试文件越来越完整。对开发来说，测试也是很重要的，编写测试文件，无论是完善开发，还是能力提升，都很有意义。

### 2.Request类构造方法优化。
与之前版本比较：

    class Request(object):
      def __init__(self):
        ...

        r = Request() 		#实例化一个 Request类。
        r.method = 'GET'	# 初始化r的各个属性。
        r.url = url
        r.params = params
        r.headers = headers
        r.auth = _detect_auth(url, auth) 	# 权限认证，如果需要的话。
现在：

    class Request(object):
      def __init__(self, url=None, headers=dict(), files=None, method=None,params=dict(), data=dict(), auth=None, cookiejar=None):
        ...

    r = Request(method='GET', url=url, params=params, headers=headers,cookiejar=cookies, auth=_detect_auth(url, auth)) #实例化一个 Request类。

很明显，这是写法上的优化，需要学习。

## 五 v0.3.0的大版本更迭（commit id为 044252ea）
### 1.自动验证API的改变。
与之前版本比较：

	 def test_AUTH_HTTPS_200_OK_GET(self):
        auth = requests.AuthObject('requeststest', 'requeststest')
        requests.add_autoauth(url, auth)
        ...
        requests.AUTOAUTHS = []		# 测试完成之后清空AUTOAUTHS
        ...
现在：

	def test_AUTH_HTTPS_200_OK_GET(self):
        auth = ('requeststest', 'requeststest')
        ...
        requests.auth_manager.empty()	# 测试完成之后清空auth_manager
        ...

原来需要实例化一个AuthObject对象， 现在只需要一个元组作为参数就可以，使用角度来说方便了很多；
原来需要手动执行 add_autoauth(), 现在第一次auth=auth后，会自动保存（且最后清空auth的方法也不同了），实现细节见4。

### 2.send() 函数中，更智能的查询参数。
测试用例：r = requests.get('http://google.com/search?test=true', params={'q': 'test'}, headers=heads)	# 这次测试，原url中已经有参数了。
与之前版本比较：

	req = _Request(("%s?%s" % (self.url, params)), method=self.method)	# 之前的做法是，直接在url后加?params。
	# 那么这次测试的结果就是http://google.com/search?test=true?q=test，很明显这样的url是错误的。
现在：

	req = _Request(self._build_url(self.url, self._enc_data), method=self.method)

    self._build_url方法如下：
        @staticmethod
      def _build_url(url, data):
          if urlparse(url).query:
              return '%s&%s' % (url, data)
          else:
              if data:
                  return '%s?%s' % (url, data)
              else:
                  return url
          
调用了标准库 urlparse ，逻辑大概为，如果url 里面有参数，把data的key,value 以&加到后面，否则，以？加到后面。 体现了代码的健壮性。

### 3.允许文件上传和post数据同时进行。

    if self.files:
        register_openers()
        if self.data:
            self.files.update(self.data)
    	
这个没什么好说的，就是把data 加到file。

### 4.新的认证管理系统。
它：可以接受形如元组的参数；第二次对同一个url 发送请求时，无需再次添加验证。
（1）首先看Request类中关于Auth的细节:

    class Request(object):
        ...
        if isinstance(auth, (list, tuple)):
          auth = AuthObject(*auth)
      if not auth:	# auth原值是None，若没有传入auth 参数时，会执行下一行代码。
          auth = auth_manager.get_auth(self.url)	# auth_manager，见（2）；get_auth方法，见（3）。
      self.auth = auth
 
也就是说，如果没有提供auth 参数时，会去 auth_manager 中拿取当前url 的认证信息。

（2）auth_manager = AuthManager()	# 每次import requests时，会产生一个auth_manager，它是一个全局的 AuthManager() 对象。

    class AuthManager(object): # 并且这个类采用了单例的写法，确保不同文件import requests 时，只有一个 auth_manager。
      def __new__(cls):		# 在调用 __init__() 方法之前，Python 首先调用 __new__() 方法
          singleton = cls.__dict__.get('__singleton__') # 获取 singleton
          if singleton is not None:
              return singleton	# 若有，则直接返回 singleton

          cls.__singleton__ = singleton = object.__new__(cls)	# 若没有 singleton ，则会调用__init__方法。

          return singleton

      def __init__(self):
          self.passwd = {}
          self._auth = {}		# dict结构，用于保存键值对

这是pythoh中，单例模式的写法。好好学。

（3）get_auth方法。        

    class AuthManager(object):
        ...
        def get_auth(self, uri):
        uri = self.reduce_uri(uri, False) # 继续调用reduce_uri方法，解析url，获得（域名＋端口， 路径），见（4）。
        return self._auth.get(uri, None) # 将（域名＋端口， 路径）作为键，获取它对应的auth。
       ...
（4）reduce_uri方法。    

    class AuthManager(object):
        ...
        def reduce_uri(self, uri, default_port=True):	# 解析url
        parts = urllib2.urlparse.urlsplit(uri)
        if parts[1]:
            scheme = parts[0]
            authority = parts[1]
            path = parts[2] or '/'
        else:
            scheme = None
            authority = uri
            path = '/'
        host, port = urllib2.splitport(authority)
        if default_port and port is None and scheme is not None:
            dport = {"http": 80,
                     "https": 443,
                     }.get(scheme)
            if dport is not None:
                authority = "%s:%d" % (host, dport)
        return authority, path	# 最后返回的其实是 （域名＋端口， 路径）
        ...
相关方法：

      def add_auth(self, uri, auth):
          uri = self.reduce_uri(uri, False)	# 解析url，获得 （域名＋端口， 路径）
          self._auth[uri] = auth # 将（域名＋端口， 路径）作为键，auth作为值，保存到_auth这个dict中。