### 组件一览
+ hacking        
一组flake8插件，用于静态检查。  
https://pypi.python.org/pypi/hacking 
+ coverage        
衡量python代码覆盖率的工具。可以单独执行/API方式或者以nose插件方式运行“nosetests --with-coverage”。  
https://nose.readthedocs.org/en/latest/plugins/cover.html  
+ discover   
测试用例发现。（2.7已经包含在unittest中，2.4需要backport) 主要在run_test.sh下使用。  
https://pypi.python.org/pypi/discover/0.4.0   
+ feedparser   
使用python进行parse RSS订阅内容主要在version API的测试中使用（versionAPI支持atom格式返回信息）
+ MySQL-python  
mysql接口的python实现
+ psycopg2  
postgresql接口的python实现
+ pylint       
对python进行静态分析、检查的工具
+ python-subunit   
subunit是测试结果的流协议。python-subunit是它的python实现。
+ sphinx      
文档生成工具（基于Restructed格式）
+ oslosphinx  
openstack对sphinx的扩展
+ testrepository  
测试结果的数据库。主要在覆盖率测试时使用。
+ mock       
对所测试的函数的外部依赖函数进行模拟替换。3.3以后已经是python标准库。
mock的实现原理也很简单，一般使用类似mokey patch的方式实现。  
+ mox        
基于java的easymock提供的python mock对象框架（基本上已经停止维护）
+ testtools   
对python标准单元测试框架的扩展。为什么使用？
  + 更好的断言    比如支持assertThat扩展
  + 更详细的debug信息  比如支持addDetails的信息
  + 扩展的同时保持兼容性  
  + python多版本的兼容性
+ tox   
通用的虚拟环境管理和测试命令行工具。


### 2