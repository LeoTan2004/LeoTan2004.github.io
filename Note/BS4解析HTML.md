# BS4解析HTML

**BS4**是一个外部的工具，需要通过*pip*引入使用，具体操作可百度

## BS4使用前需要进行的操作

```python
from bs4 import BeautifulSoup
def get_bs(url,timeout):#请求一个网页，以HTML的格式返回，如果超时或者网页我无法访问，则返回一个None
    header={
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/107.0.0.0 Safari/537.36 Edg/107.0.1418.35"}
    request = urllib.request.Request(url=url,headers=header)
    try:
        response = urllib.request.urlopen(request,timeout=timeout)
        html = response.read().decode("utf-8")
    except Exception:
        return None
    bs= BeautifulSoup(html,"html.parser")#bs就是已经初始化好的对象
    
```

经过这一步是，我们已经得到了一个***BeautifulSoup***的对象了。

## BS4抓取数据

#### 父子元素直接访问

html经过解析后形成了一个树状结构。例如：

`bs.title`就可以直接获取HTML中的标题。也就是获取了<title>的标签；

`bs.a`可以获取HTML中的第一个<a>的标签；

`bs.javascript`可以获取HTML中的第一个<javascript>的标签。

> 但是要注意，这里获取的标签是包括子标签的，并且注意，这个标签并不是普通的字符串，而是一个对象，也是一个**TAG**类的实例。我们可以通过这个对象访问它的父和子元素
>
> 访问tag对象的父元素：**tag.parent**
>
> 访问tag对象的子元素：**tag.contents**
>
> *PS:子元素不一定只有一个，因此返回值是一个列表，不可以直接使用，可以遍历或者通过下标访问*

如果你试着`print(bs.title)`的话，你可能对看到一下结果：

> C:\\\<title>这是标题<title>

没错，如果直接输出TAG对象的话，Python会把他的标签连同内容一起输出（可能还有子元素）,也就是从开始标签的一直到结束标签的全部输出。那么如果只想要标签里面的数据呢，怎么办？

BS4当然有办法，对于TAG对象来讲，其中的内容也是他的一个属性，那就直接访问咯！

> TAG.string

这便是TAG对象的内容，这里面也不会有子元素。可以***难道这个就是一个stirng数据吗？***我们可以使用`type()`函数来看一看？`print(TAG.string)`让我们看看他到底是什么类型。

> <class ‘bs4.element.NavigableString’>

是的，他并不是一个string类型，仍然是**element中的一个小类**，因此咱们还可以使用`NavigableString.parent`来获取他的父元素，也就是那个***TAG***

## BeautifulSoup4中的类

###  Element

这是所有类的父类

##### 属性

- **name**：标签名称，其实也就是标签是什么类型，例如对于`<title>`标签，他的**name**就是*title*
- **parent**：父元素（树状结构）
- **contents**：子元素（list结构）

### TAG

TAG类是BeautifulSoup包中所有的类，他是在文档被解析后所生成的类

##### 属性

- **string**：元素中的内容，一般也就是我们需要抓取的数据
- **attrs**：元素的属性，因为属性不一定只有一个，因此返回的是个字典（dict），也就是**键值对组**，就相当于*maps<keys,values>*，比如`bs.a.attrs`可以获取他的``href、class`，不过可以才想到，他应该和上面的**stirng**一样，也是一个对象，可以通过`href.parent`访问他的父元素，也就是属性所有者，这便是**树状结构**的体现。

### BeautifulSoup

##### 属性

- **title**：就是document
- 其中包含所有数据，可以直接访问，例如`beautifulsoup.a`,可以拿到第一个a标签。

## 节点获取（条件节点）

### 1. find_all()

查找所有，比通过树状结构访问获得的元素更多

- 通过字符串查找：

  `find_all("a")`找到所有`a`的标签

  还可以使用列表：`find_all(["a","b"])`会将所有a，b标签返回

- 正则表达式查找：

  `find_all(re.compile("a"))`查找所有标签中含有a的元素，比如`head`、`meta`等当然不止这些，如果他的属性里面也含有*a*也会被包含进来，比如`href`之中如果含有a字样，也会包含。

- 自定义查找：传入的不是参数，而是函数，没错，要知道python可是c语言家族的，所以函数做参数也无可厚非，那应该怎么做呢？

  ```python
  def name_is_exist(tag):
      return tag.has_attr("name")
  beautiful.find_all(name_is_exist)
  #这样就会把所有传入并返回true的所有标签放回（list），但是就上面这个算法，还有个更优的方法
  beautiful.find_all(name=True)
  ```

- 进阶正则表达式（*kwargs*）：上面一个正则表达式的查找是一种十分无脑的东西，因为他会无脑的在两个尖括号里面直接查找，没有限制；如果我们想指定哪个属性的值符合正则怎么办？

  ```python
  beautiful.find_all(class_=re.compile("\d"))
  #可以查找所有类名中含数组的标签
  ```

- 限定搜索：

  使用`find_all()`时传参`limit=4`

### 2.css选择器（select）

```python
beautiful.select("div.title")
#返回所有div下的title的标签
beautiful.select(".display")
#返回所有display类的标签
beautiful.select("#username")
#返回所有id为username的标签
beautiful.select("a[class='bri']")
#返回所有class为bri的a标签
beautiful.select("div > a")
#返回所有div下的a标签
beautiful.select("#cs ~ ul")
#返回所有和id为cs的标签同一级的ul标签
#详细语法见css选择器
```

