---
layout: post
title: "Python unicode and bytestring revisit"
---

### Unicode

* unicode的字符集几乎包含了世界上所有文字，它用code point来表示每一个字符
* utf8 encoding是一种unicode的编码方法
* utf8使用1-4个字节来存储unicode字符，utf16使用2-4个字节，而utf32使用定长的4个字节
    
### Bytestring

在Python中用单引号，双引号或者三引号括起来的字符都是str（确切地说是str的string literal），它们的实际值，在不同的系统不尽相同（ASCII码例外）。

在windows下

{% codeblock lang:python %}
>>> ren = "人"
>>> ren
'\xc8\xcb'
>>> ren.decode("cp936")
u'\u4eba'
>>> print ren.decode("cp936")
人
{% endcodeblock %}

而在ubuntu下

{% codeblock lang:python %}
>>> ren = "人"
>>> ren
'\xe4\xba\xba'
>>> ren.decode("utf8")
u'\u4eba'
>>> print ren.decode("utf8")
人
{% endcodeblock %}

可见，对于str，Python使用了系统的默认编码（而不是Python默认的ascii编码）进行编码

### Confusion

Why u"你好" works while "你好" doesn't?

{% codeblock lang:python %}
>>> "你好".encode("utf8")
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
UnicodeDecodeError: 'ascii' codec can't decode byte 0xe4 in position 0: ordinal not in range(128)

>>> u"你好".encode("utf8")
'\xe4\xbd\xa0\xe5\xa5\xbd'
{% endcodeblock %}

首先，对于**encode**操作，错误里报"..can't **decode**.."，这个追过代码没追到，在Stackoverflow上提问，有同学帮忙解决了，看[这里](http://stackoverflow.com/a/9644206/523517)。


其次，对于为什么同是对literal进行encode， unicode literal就不会报错呢。


在[What’s New In Python 3.0](http://docs.python.org/release/3.0.1/whatsnew/3.0.html#text-vs-data-instead-of-unicode-vs-8-bit)找到了答案：

>The biggest difference with the 2.x situation is that any attempt to mix text and data in Python 3.0 raises TypeError, whereas if you were to mix Unicode and 8-bit strings in Python 2.x, it would work if the 8-bit string happened to contain only 7-bit (ASCII) bytes, but you would get UnicodeDecodeError if it contained non-ASCII values. This value-specific behavior has caused numerous sad faces over the years.

这么说来还是Python 2.X自身的问题了。在2.X的版本中，string和bytes使用的数据类型都是str，所以str既是字面值(literal)，也是二进制值(binary)。

{% codeblock lang:python %}
>>> '\x61'
'a'
{% endcodeblock %}

而且很容易造成混淆的是ASCII码的字面值和二进制值都是用字面值表示

{% codeblock lang:python %}
>>> 'a'
'a'
{% endcodeblock %}

这对于ASCII码来说无所谓，但如果str包含了非ASCII码的字符，在encode的时候就会报上述错。而在Py3k中，这两者被区分开了

>Python 3.0 uses the concepts of text and (binary) data instead of Unicode strings and 8-bit strings. All text is Unicode; however encoded Unicode is represented as binary data. The type used to hold text is str, the type used to hold data is bytes. 

>You can no longer use u"..." literals for Unicode text. However, you must use b"..." literals for binary data.

在Py3k下运行的结果

{% codeblock lang:python %}
>>> u"人"
File "<stdin>", line 1
u"人"
	 ^
SyntaxError: invalid syntax
>>> "人"
'人'
>>> type("人")
<class 'str'>
>>> "人".encode()
b'\xe4\xba\xba'
>>> b'\xe4\xba\xba'.decode()
'人'
>>> '\xe4\xba\xba'.decode()
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
AttributeError: 'str' object has no attribute 'decode'
{% endcodeblock %}

在[Django的字符串处理](https://docs.djangoproject.com/en/dev/ref/unicode/#general-string-handling)中，Django假设所有的str都使用了UTF8编码，如果在某些涉及到中文的地方使用了str就很容易造成错误，所以在使用字符串这种类型的时候，最好统一使用unicode。

最后，可以通过更改解释器的默认编码来避免上述问题，当然这种做法已经deprecated了。

{% codeblock lang:python %}
>>> import sys
>>> sys.getdefaultencoding()
'ascii'
>>> reload(sys)
<module 'sys' (built-in)>
>>> sys.setdefaultencoding("utf8")

>>> "人"
'\xe4\xba\xba'

>>> "人".encode()
'\xe4\xba\xba'

>>> str(u"人")
'\xe4\xba\xba'

>>> unicode(str("人"))
u'\u4eba'
{% endcodeblock %}
