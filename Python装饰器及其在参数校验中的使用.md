# Python装饰器及其在参数校验中的使用


本文主要介绍Python装饰器的基本原理以及一个装饰器的应用实例：参数校验。
全文分为三章：
1. 基本原理：这部分是正确理解装饰器所需要的基础理论，但没有引入任何装饰器的话题。
2. 装饰器：根据第一章说阐述的基础技术，一步一步把装饰器建立并完善，最后得到一个可以解决实际问题的装饰器类框架。
3. 应用实例：一个装饰器的参数校验框架。首先是一个最基本的版本，然后介绍了现成的参数校验方案validator.py，最后是集成了validator.py的参数校验装饰器框架。
## 基本原理

### 函数定义
装饰器是作用在函数上的，所以我们先重温一下Python的函数，例如下面这个很简单的函数。这个函数能打印一个打招呼的字符串。
```python
def greet(name='World'):
    print('Hello, {}'.format(name))
    
greet()
```
 输出示例：
```shell
Hello, World
```
### 函数对象
**一切皆对象**。在Python里面，函数也是一个对象实例，所以我们可以看看上面那个hello函数的类型，属性和方法列表以及内存地址。甚至可以把它赋值给另外一个变量，然后通过另一个变量调用它。
```python
def greet(name='World'):
    print('Hello, {}'.format(name))

#正常的函数调用
greet()

#打印hello函数实例
print(greet)
#打印hello函数实例类型
print(type(greet))
#打印hello函数实例的属性和方法列表
print(dir(greet))
#打印hello函数实例的id
print(id(greet))

#把hello函数实例赋值给一个变量hi
hi = greet
#打印hi变量所指的地址
print(id(hi))
#通过hi调用hello函数
hi()
```
 输出示例：
 ```python
Hello, World
<function greet at 0x000001A742FBC1E0>
<class 'function'>
['__annotations__', '__call__', '__class__', '__closure__', '__code__', '__defaults__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__get__', '__getattribute__', '__globals__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__kwdefaults__', '__le__', '__lt__', '__module__', '__name__', '__ne__', '__new__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__']
1817894961632
1817894961632
```

### 函数作为参数
一个函数既然是一个对象实例，那么函数也可以作为函数的参数，例如下面hello和hi两个函数是以不同的方式打招呼，而greet函数则是一个通过参数输入一个函数的方式实现不同的具体打招呼姿势。这样有趣的事情又再次发生了，是不是感觉到程序有了一点灵性，似乎活起来了：
```python
def hello(name):
	print('Hello, {}'.format(name))

def hi(name):
	print('Hi, {}'.format(name))

def greet(func,name='World'):
	return func(name)

print(greet(hello,'World'))
print(greet(hi,'Derek'))
```

输出示例：
```python
Hello, World
Hi, Derek
```
### 函数嵌套
为了说明下面的例子，先说说函数嵌套，简单来说，函数嵌套就是在函数里面定义函数（准确来说，如果内部函数引用了外部自由变量，还会形成闭包）。
下面是一个函数嵌套的例子：hello函数还是之前的打招呼函数，这次在他里面定义了一个something_else函数。这个something_else函数的功能就是打印一句没头没脑的话。
可以看到something_else函数只能在hello函数内部被访问，要是在hello函数外部访问，就会出现NameError的错误。
其实两者在业务逻辑上没啥关系，greet函数仅仅在其内部定义了something_else函数，并没有调用。而外部也无法调用something_else，在这个情况下，函数的嵌套的作用并没有体现出来。
```python
def greet(name='World'):
    def something_else():
	    	print('something_else called.')
    print('Hello, {}'.format(name))
    
greet()
something_else()
```
输出示例：
```python
Traceback (most recent call last):
Hello, World
  File "C:/Users/derek/PycharmProjects/untitled/deco.py", line 9, in <module>
    something_else()
NameError: name 'something_else' is not defined
```
### 函数作为返回值
上面的例子函数嵌套没有什么实际用处，那他的作用体现在哪里呢？函数嵌套一个常用用途就是把内部的函数作为返回值返回给外部。
因为既然是一个对象实例，那么它也可以作为返回值被另外一个函数返回。那么在这个情况下，有趣的事情就发生了。
```python
def greet(mode, name='World'):
	def hello():
		print('Hello, {}'.format(name))

	def hi():
		print('Hi, {}'.format(name))

	if mode == 'hello':
		return hello
	else:
		return hi

say_hello = greet(mode='hello')
say_hi = greet(mode='hi',name='Derek')

say_hello()
say_hi()
```
输出示例：
```python
Hello, World
Hello, Derek
```

## 装饰器
至此，装饰器的前置理论基础都已经过了一遍。可以正式开讲装饰器的实现了。

### 基本定义
装饰器的功能是能在不修改原有函数的代码以及不改变原有调用接口的情况下部分地改变的原有函数的逻辑。从这个描述可以看到，装饰器有几个关键要素：
1. 不改变原有代码
2. 不改变原有调用接口
3. 实现对原有函数逻辑的修改

具体的实现就是之前介绍过的基本原理的组合运用了。
如果仅仅是在不改变原有函数的代码的情况下部分修改原有函数的逻辑，其实不需要装饰器也能做到：大体的思路是把原有函数作为参数输入到封装函数里面，然后封装函数在执行完前置的装饰逻辑后，调用原有函数，然后再执行后置装饰逻辑，完整整个装饰过程。例如，下面的例子是在说完Hello的正常情况之后，再多说一句"Have a nice day!"
```python
def hello():
	print('Hello, World')


def have_a_nice_day(func):
	func()
	print('Have a nice day!')

have_a_nice_day(hello)
```
输出示例：
```python
Hello, World
Have a nice day!
```
但从上面的例子可以看到，hello函数是被装饰的函数，从输出来看的确能实现在原有的打招呼逻辑后添加一个问候语"Have a nice day!"。但现在调用这个打招呼已经不是通过被装饰函数hello()了，而是通过装饰函数have_a_nice_day()。这就相当于改变了的调用接口了。
那么怎么实现不改变原有的调用接口，想想之前说过的基础技术原理，如果我们把函数作为对象进行赋值，修改一下hello变量的指向，让他指向have_a_nice_day所指的函数对象，不就在不修改原有代码并保持原有的调用接口的情况下，实现了修改原有逻辑的功能么？
```python
def hello():
	print('Hello, World')

def have_a_nice_day(func):
	func()
	print('Have a nice day!')

temp = hello
hello = have_a_nice_day
hello(temp)
```
输出示例：
```python
Hello, World
Have a nice day!
```
但是这个方案，明眼人都能看出问题，实现起来太丑了啊！有没有优雅的方式呢？再想想之前的说过的基础技术原理，如果我们把函数嵌套和把函数作为返回值用上呢？如下所示：
```python
def hello():
	print('Hello, World')

def have_a_nice_day(func):
	def wrap_func():
		result = func()
		print('Have a nice day!')
		return result
	return wrap_func

hello = have_a_nice_day(hello)
hello()
print(hello)
```
输出示例：
```python
Hello, World
Have a nice day!
<function have_a_nice_day.<locals>.wrap_func at 0x00000125BC474AE8>
```
这个方案似乎已经很完美的解决了之前的问题，但是这个时候其实还是有一个问题：可以看到，现在hello所指的函数其实已经变成了have_a_nice_day的局部的wrap_func函数了（元信息发生了改变）。也就是说我们还是能发现他其实不是原来的那个函数了。那这个问题怎么解决呢？我们先说一下装饰器的@符号语法糖吧。

### @语法糖
Python提供了@符号作为装饰器的语法糖，在定义被装饰函数的时候可以同时使用@符号注解，这样被装饰函数就会自动带上装饰逻辑：
```python
def have_a_nice_day(func):
	def wrap_func():
		a = func()
		print('Have a nice day!')
		return a
	return wrap_func

@have_a_nice_day
def hello():
	print('Hello, World')

hello()
print(hello)
```
输出示例：
```python
Hello, World
Have a nice day!
<function have_a_nice_day.<locals>.wrap_func at 0x00000248EB724AE8>
```
其实这个@注解就相当于执行了f赋值操作：
```python
hello = have_a_nice_day(hello)
```

### functools.wraps
可以看到即使使用了@符号，依然没有解决原有函数变量的类型发生变化的情况。但其实Python早就提供了现成的解决方案：functools.wraps
```python
from functools import wraps


def have_a_nice_day(func):
	@wraps(func)
	def wrap_func():
		a = func()
		print('Have a nice day!')
		return a

	return wrap_func


@have_a_nice_day
def hello():
	print('Hello, World')


hello()
print(hello)
```
输出示例：
```python
Hello, World
Have a nice day!
<function hello at 0x0000022176DA4AE8>
```
可以看到现在hello函数即使被装饰了，依然还是hello函数（保持了元信息）。
很明显functools.wraps本身也是一个装饰器，查看他的源码，发现它把原函数的元信息拷贝到装饰器函数中，这使得装饰器函数也有和原函数一样的元信息。

### 被装饰函数参数
至此，最基础最核心的装饰器的技术介绍完毕了。
但是这样的装饰器并不能解决实际问题，因为我们之前被装饰的函数都是无参数的，如果是带参数的函数该如何被装饰呢？要回答这个问题，其实需要从浅入深地回答两个问题：
1. 参数是如何通过装饰器传入到被装饰函数
2. 装饰器是如何以统一的方式装饰有不同参数列表的函数

假设我们现有一个打招呼的hello()函数，他的输入参数是一个名字name。他会根据name打印相应的打招呼字符串。
```python
def hello(name):
	print('Hello, {}'.format(name))

hello('Derek')
```
输出示例：
```python
Hello, Derek
```
再假设，我们还是需要在原有的打招呼字符串后面再打印一句"Have a nice day!"。
那么装饰器要怎么才能提供name这个参数接口，供被装饰后的函数使用呢？其实很简单：只要我们在装饰器的内部函数中和被装饰函数使用相同的参数列表就可以了。如下所示：
```python
from functools import wraps

def have_a_nice_day(func):
	@wraps(func)
	def wrap_func(name):
		result = func(name)
		print('Have a nice day!')
		return result

	return wrap_func

@have_a_nice_day
def hello(name):
	print('Hello, {}'.format(name))

hello('Derek')
```
输出示例：
```python
Hello, Derek
Have a nice day!
```

很好，现在问题一**参数是如何通过装饰器传入到被装饰函数**解决了！
真的吗？这样的解决方案有什么问题呢？问题在于这个装饰器就只能装饰这种只带一个字符串参数的函数了，例如下面还有另外一个版本打招呼函数hello_long()，他接收两个参数，一个是对方的名字name，一个是自己的名字my_name。既和对方打招呼，又介绍了自己。
```python
def hello_long(name, my_name):
	print('Hello, {}'.format(name))
	print('I am {}'.format(my_name))
```
这个时候如果还是使用之前的装饰器来装饰hello_long()函数，就会扑街了：
```python
from functools import wraps

def have_a_nice_day(func):
	@wraps(func)
	def wrap_func(name):
		result = func(name)
		print('Have a nice day!')
		return result
	return wrap_func

@have_a_nice_day
def hello(name):
	print('Hello, {}'.format(name))

@have_a_nice_day
def hello_long(name, my_name):
	print('Hello, {}'.format(name))
	print('I am {}'.format(my_name))

hello('Derek')
hello_long('Meredith', 'Derek')
```
输出示例：
```python
Traceback (most recent call last):
  File "C:/Users/derek/PycharmProjects/untitled/deco.py", line 25, in <module>
    hello_long('Derek','Meredith')
Hello, Derek
TypeError: wrap_func() takes 1 positional argument but 2 were given
Have a nice day!
```

所以这就是第二个问题了：**装饰器是如何以统一的方式装饰有不同参数列表的函数**。
其实这个的问题实质是**装饰器中的内部函数要怎么定义函数参数列表才能处理各种不定长度的参数**。
Python不是早就提供了现成的解决方案了吗？可变参数和关键字参数： **(\*args, \*\*kwargs)** 。
如下所示：
```python
from functools import wraps

def have_a_nice_day(func):
	@wraps(func)
	def wrap_func(*args,**kwargs):
		result = func(*args,**kwargs)
		print('Have a nice day!')
		return result
	return wrap_func

@have_a_nice_day
def hello(name):
	print('Hello, {}'.format(name))

@have_a_nice_day
def hello_long(name, my_name):
	print('Hello, {}'.format(name))
	print('I am {}'.format(my_name))

hello('Derek')
hello_long('Meredith', 'Derek')
```
输出示例：
```python
Hello, Derek
Have a nice day!
Hello, Meredith
I am Derek
Have a nice day!
```
可以看到，现在这个版本的装饰器已经可以同时处理有着一个和两个参数的打招呼函数了。
### 装饰器参数
很好，问题都解决了，这样的装饰器已经可以解决现实问题了。
慢着，现实是很残酷的！例如在实际问题中装饰行为可能要根据不同的函数而有所区别。那么怎么实现这种装饰行为的区别呢？
别忘了装饰器其实就是一个函数，既然是函数，那就有参数列表。函数内部的逻辑控制就是通过参数来实现的。
例如，我们之前的打招呼装饰器需要做得更智能了，当天气好的时候打印"Have a nice day!"，但是天气不好的时候就打印"What a bad day!"。我们可以简单为装饰器引入一个名为good_weather的bool类型的参数用来控制不同的打印逻辑。
```python
from functools import wraps

def have_a_nice_day(func, good_weather):
	@wraps(func)
	def inner_wrap_func(*args, **kwargs):
		result = func(*args, **kwargs)
		if good_weather:
			print('Have a nice day!')
		else:
			print('What a bad day!')
		return result
	return inner_wrap_func

def hello(name):
	print('Hello, {}'.format(name))

def hello_long(name, my_name):
	print('Hello, {}'.format(name))
	print('I am {}'.format(my_name))

hello = have_a_nice_day(good_weather=True, func=hello)
hello_long = have_a_nice_day(good_weather=False, func=hello_long)

hello('Derek')
hello_long('Meredith', 'Derek')
print(hello)
print(hello_long)
```
输出示例：
```python
Hello, Derek
Have a nice day!
Hello, Meredith
I am Derek
What a bad day!
<function hello at 0x0000021609824BF8>
<function hello_long at 0x0000021609824C80>
```
很好，又实现了一个更加高级的功能。但根据前面的尿性，这个时候，肯定还是有问题的。问题是什么呢？
我们其实并没有使用@符号的装饰器语法糖来实现这个功能的。为什么不呢？因为如果装饰器是这么写的话，是不能用于语法糖了。@符号是等价于一个参数列表为一个函数参数的装饰器函数作用在被装饰函数上的赋值运算。现在装饰器函数参数列表已经不是仅有一个函数参数了，而是引入了其他各种各样的参数。这个时候@符号无法转换成正确的等价赋值运算了。
那该如何解决呢？
现在可以公开的情报：
1. @符号语法糖必须作用在参数列表仅含有一个函数变量的函数上：
**wrap_function(func)**
2. 必须在嵌套的内部函数中带上可变参数以装饰有着不同参数列表的被装饰函数:
**wrap_function(\*args,\*\*kwargs)**
3. 装饰器函数还需要有着自己的参数列表
**decorator(customized_arg1,customized_arg2,customized_arg3,...)**

之前的方式之所以不能同时满足上述三点，是因为原来的结构只有一层嵌套，无法分离这三点要素。
但是再增加一层嵌套呢？，例如：
```python
def decorator(customized_arg1, customized_arg2, customized_arg3, ...):
	def outer_wrap_func(func):
		@wraps(func)
		def inner_wrap_func(*args, **kwargs):
			# pre_decoration
			result = func(*args, **kwargs)
			# post_decoration
			return result
		return inner_wrap_func
	return outer_wrap_func
```
这里的最外层decorator就是装饰器自身的接口，定义了他本身能接收什么怎样的参数来实现装饰逻辑，最外层返回的是中间层定义的outer_wrap_func()函数变量；
中间层outer_wrap_func函数变量的参数接口是一个函数对象func，他返回了最里层定义inner_wrap_func()函数变量。其实来到这里就和上一个小节的处理逻辑完全一样了。
最里层就是inner_wrap_func()的参数列表是可变参数，就是用来适配有着各种不同参数的被装饰函数的。
这里的逻辑用链式表达就是：
**decorator(customized_arg1, customized_arg2, customized_arg3, ...)(func)(\*args,\*\*kwargs)**
下面就举个例子：
```python
from functools import wraps


def have_a_nice_day(good_weather):
	def outer_wrap_func(func):
		@wraps(func)
		def inner_wrap_func(*args, **kwargs):
			result = func(*args, **kwargs)
			if good_weather:
				print('Have a nice day!')
			else:
				print('What a bad day!')
			return result

		return inner_wrap_func

	return outer_wrap_func


@have_a_nice_day(good_weather=True)
def hello(name):
	print('Hello, {}'.format(name))


@have_a_nice_day(good_weather=False)
def hello_long(name, my_name):
	print('Hello, {}'.format(name))
	print('I am {}'.format(my_name))


hello('Derek')
hello_long('Meredith', 'Derek')
print(hello)
print(hello_long)
```
输出示例：
```python
Hello, Derek
Have a nice day!
Hello, Meredith
I am Derek
What a bad day!
<function hello at 0x0000024CF4BE4B70>
<function hello_long at 0x0000024CF4BE4C80>
```

### 装饰器类
至此，我们就真的可以利用上述的装饰器技术来解决实际问题了。
但是我们还是再来一个锦上添花的内容吧：**装饰器类**。
这个其实是很自然的，函数是对象，那么为什么不能就创建一个类，并用这个类来实例化一个对象实现函数能实现的功能呢？这只需要实现类的 **\_\_call\_\_()** 方法就可以了。
如果只要我们把之前的装饰函数在__call__()方法中实现，就能创建一个装饰器类，而之前带参数的装饰器的参数也可以通过类的成员变量来实现。
例子在此：
```python
from functools import wraps

class DayGreeting():

	def __init__(self,good_weather):
		self._good_weather = good_weather

	def __call__(self, func):
		@wraps(func)
		def wrap_func(*args, **kwargs):
			result = func(*args, **kwargs)
			if self._good_weather:
				print('Have a nice day!')
			else:
				print('What a bad day!')
			return result
		return wrap_func


@DayGreeting(good_weather=True)
def hello(name):
	print('Hello, {}'.format(name))


@DayGreeting(good_weather=False)
def hello_long(name, my_name):
	print('Hello, {}'.format(name))
	print('I am {}'.format(my_name))


hello('Derek')
hello_long('Meredith', 'Derek')
print(hello)
print(hello_long)
```
输出示例：
```python
Hello, Derek
Have a nice day!
Hello, Meredith
I am Derek
What a bad day!
<function hello at 0x0000021E4C9A4BF8>
<function hello_long at 0x0000021E4C9A4D08>
```
上面的代码就不多解释了，其实和装饰器没多少关系了，这里更多是可调用对象的知识点。

## 实际应用
上面说了那么多，能解决现实世界中的实际问题吗?
其实装饰器并不是Python特有的，它是一种设计模式，是比具体的编程语言更高级的抽象。
它允许向一个现有的对象添加新的功能，同时又不改变其结构。这种类型的设计模式属于结构型模式，它是作为现有的类的一个包装。这种模式创建了一个装饰类，用来包装原有的类，并在保持类方法签名完整性的前提下，提供了额外的功能。
它的这种逻辑很明显是添加在原有逻辑的前面和后面，带有明显的切面特性，所以也叫面向切面编程。日常也有很多面向切面的场景，例如插入日志、性能测试、事务处理等。装饰器是解决这类问题的绝佳设计，有了装饰器，我们就可以抽离出大量函数中与函数功能本身无关的雷同代码并继续重用。
我们这里就用装饰器来实现参数校验的功能。
### 功能需求
我们编程各种程序，里面都会带有大量的函数。函数业务逻辑难免会有漏洞，如果函数的参数取值不合理，就会引发Bug。
所以我们需要在函数内部业务代码真实运行之前，执行参数的校验，如果校验成功，就执行业务逻辑；如果校验失败就抛出异常，终止执行。
很自然的选择，一般会采用一堆条件分支if-else来进行判断，这种方式有很多优点也有很多缺点：
* 优点：
实现简单，判断条件完全自定义，可以实现很复杂的判断规则；
* 缺点：
对于不同函数的参数，大多数的检验要求都类似的，例如都是判断是不是空，文件是否存在，数字型参数是否在一个合理范围。每个函数在执行实际业务逻辑之前都有一大段的参数检验代码块会影响程序的 

而从上述的描述来看，参数校验其实就是一个切面，装饰器能很好的解决这个问题。参数校验装饰器需要:
1. 提供一个统一的校验切面，切入到每个需要进行参数校验的函数中；
2. 校验成功是继续执行业务逻辑；校验失败就抛出异常，终止执行；
3. 针对不同函数的不同参数在不同的校验失败的情况下抛出不同的异常；
4. 同时适配所有的拥有不同参数列表的函数；
5. 提供可扩展的校验逻辑，以便适配各种不同的实际业务需求。

### 基本框架
根据前面已经说过的基本技术原理，这个校验框架的设计如下：
1. 采用装饰器类实现；
2. 实现一个方法从输入的被装饰函数中提取所有的参数键值对;
3. 装饰逻辑（参数检验逻辑）在__call__()方法中实现；
4. \_\_call__()方法的参数是函数类型变量，里面定义的嵌套函数接受可变参数，适配各种不同参数列表被校验函数；
5. 检验规则使用类属性定义，例如是否需要判断非空，需要判断的数值范围...

因为规则太多，所以下面的例子仅仅是校验参数是不是为空，如果需要增加其他的校验规则，则需要添加对应的属性和校验逻辑。程序的大体的结构如下：

```python
from functools import wraps
from collections import OrderedDict


class ArgsValidator():

	def __init__(self, check_none_list=None):
		'''add attributes for other validtion purpose'''
		self._check_none_list = check_none_list

	def get_func_sorted_kwargs(self, func, args, kwargs):
		sorted_kwargs = OrderedDict()
		all_arg_names = func.__code__.co_varnames
		for arg_num, arg_value in enumerate(args):
			sorted_kwargs[all_arg_names[arg_num]] = arg_value
		for arg_name in all_arg_names:
			if arg_name in kwargs:
				sorted_kwargs[arg_name] = kwargs[arg_name]
		return sorted_kwargs

	def __call__(self, func):
		@wraps(func)
		def wrap_func(*args, **kwargs):
			sorted_kwargs = self.get_func_sorted_kwargs(func=func, args=args, kwargs=kwargs)
			if self._check_none_list:
				for kwargs_key in sorted_kwargs:
					if kwargs_key in self._check_none_list:
						if sorted_kwargs[kwargs_key]:
							print('Attribute {} is not none, passed'.format(kwargs_key))
						else:
							print('Validating attribute {} is none, failed'.format(kwargs_key))
							raise self._check_none_list[kwargs_key](
								'Validating attribute {} is none, failed'.format(kwargs_key))
			# other validation logic implement here
			result = func(*args, **kwargs)

			return result

		return wrap_func


@ArgsValidator(check_none_list={'a': ValueError})
def test_func1(a, b, c):
	print(a)
	print(b)
	print(c)


if __name__ == '__main__':
	test_func1(1, 2, 3)
	{'a': 1, 'b': 2, 'c': 3}
	test_func1(1, b=2, c=3)
	{'a': 1, 'b': 2, 'c': 3}
	test_func1(b=1, a=2, c=3)
	{'a': 2, 'b': 1, 'c': 3}

	test_func1(4, 2, c=3)
	test_func1(None, 2, c=3)
```
输出示例：
```python
Attribute a is not none, passed
4
2
3
Validating attribute a is none, failed
Traceback (most recent call last):
  File "C:/Users/derek/PycharmProjects/untitled/closure.py", line 54, in <module>
    test_func1(None, 2, c=3)
  File "C:/Users/derek/PycharmProjects/untitled/closure.py", line 37, in wrap_func
    'Validating attribute {} is none, failed'.format(kwargs_key))
ValueError: Validating attribute a is none, failed
```
这里解释一下上面的程序是怎么解决前面的定下来的几点要求：
1. 采用装饰器类实现:
定义了装饰器类**ArgsValidator**
2. 实现一个方法从输入的被装饰函数中提取所有的参数键值对:
定义并实现了方法：**get_func_sorted_kwargs(self, func, args, kwargs)** ，此方法的作用地位相当关键，因为即使针对同一个函数被调用时，输入的参数格式，顺序都是不固定的，例如下面这个简单的函数，下面这些都是合法的调用形式：
```python
def test_func1(a, b, c):
	print(a)
	print(b)
	print(c)

test_func1(1, 2, 3)
test_func1(1, b=2, c=3)
test_func1(b=1, a=2, c=3)
```
get_func_sorted_kwargs方法就是根据输入的函数，以及函数被调用时输入的参数整理排序得出有序的函数的关键字参数列表形式的参数字典。针对上面几种情况，这个方法分别返回：
```python
test_func1(1, 2, 3)
{'a': 1, 'b': 2, 'c': 3}
test_func1(1, b=2, c=3)
{'a': 1, 'b': 2, 'c': 3}
test_func1(b=1, a=2, c=3)
{'a': 2, 'b': 1, 'c': 3}
```
这样后续的校验逻辑就可以字啊一个统一的基础上开展了。
3. 装饰逻辑（参数检验逻辑）在__call__()方法中实现：
这点没什么特别，前面的小节已经解释过了。
4. \_\_call__()方法的参数是函数类型变量，里面定义的嵌套函数接受可变参数，适配各种不同参数列表被校验函数；
这点也没什么特别，前面的小节已经解释过了。
5. 检验规则使用类属性定义，例如是否需要判断非空，需要判断的数值范围：
这点是校验逻辑的关键点，表征着这个参数校验类能支持多少种校验，当前只是支持了非空校验，如果要支持其他的校验逻辑，就需要在构造器创建对应的属性，然后在__call__()方法中逐一实现具体的校验逻辑。

### validator.py
上述已经实现可一个基本的校验框架了，但是实现具体的校验逻辑还是需要自己做，这个体力活做起来并不轻松。如果别人已经实现了校验逻辑，我们可以集成一下不就可以了吗？
如果抱着这种不要自己造轮子的思想，我搜了一下别人的参数校验实现，发现有一个挺好的框架叫**validator.py**。
他已经实现了基本所有常见的校验逻辑了，我们只需要构建一个规则字典就可以了。下面是他官方上面的例子，大家感受一下：
```python
from validator import Required, Not, Truthy, Blank, Range, Equals, In, validate

# let's say that my dictionary needs to meet the following rules...
rules = {
    "foo": [Required, Equals(123)], # foo must be exactly equal to 123
    "bar": [Required, Truthy()],    # bar must be equivalent to True
    "baz": [In(["spam", "eggs", "bacon"])], # baz must be one of these options
    "qux": [Not(Range(1, 100))] # qux must not be a number between 1 and 100 inclusive
}

# then this following dict would pass:
passes = {
    "foo": 123,
    "bar": True, # or a non-empty string, or a non-zero int, etc...
    "baz": "spam",
    "qux": 101
}
>>> validate(rules, passes)
(True, {})

# but this one would fail
fails = {
    "foo": 321,
    "bar": False, # or 0, or [], or an empty string, etc...
    "baz": "barf",
    "qux": 99
}
>>> validate(rules, fails)
(False, {
 'foo': ["must be equal to 123"],
 'bar': ['must be True-equivalent value'],
 'baz': ["must be one of ['spam', 'eggs', 'bacon']"],
 'qux': ['must not fall between 1 and 100']
})
```
 
### 集成方案
有了这么现成的参数校验逻辑实现，我们就可以把这个别人造好的轮子搬过来组装我们的汽车了。
```python
from functools import wraps
from collections import OrderedDict
from validator import Truthy,GreaterThan, InstanceOf,validate


class ArgsValidator():
	def __init__(self, rules):
		'''add attributes for other validtion purpose'''
		self._rules = rules

	def get_func_sorted_kwargs(self, func, args, kwargs):
		sorted_kwargs = OrderedDict()
		all_arg_names = func.__code__.co_varnames
		for arg_num, arg_value in enumerate(args):
			sorted_kwargs[all_arg_names[arg_num]] = arg_value
		for arg_name in all_arg_names:
			if arg_name in kwargs:
				sorted_kwargs[arg_name] = kwargs[arg_name]
		return sorted_kwargs

	def __call__(self, func):
		@wraps(func)
		def wrap_func(*args, **kwargs):
			sorted_kwargs = self.get_func_sorted_kwargs(func=func, args=args, kwargs=kwargs)
			if self._rules:
				for err_type in self._rules:
					rule = self._rules[err_type]
					val_result, val_err = validate(rule, sorted_kwargs)
					if not val_result:
						raise err_type(val_err)
			result = func(*args, **kwargs)
			return result

		return wrap_func


@ArgsValidator(rules={
	ValueError: {
		'a': [Truthy()],
		'b': [GreaterThan(1)],
		'c': [InstanceOf(int)]
	}
})
def demo_func(a, b, c):
	print('a={},b={},c={}'.format(a,b,c))


if __name__ == '__main__':
	demo_func(1, 2, 3)
	demo_func(4, 2, c=3)
	demo_func(None, 2, c=3)
	#ValueError: {'a': ['must be True-equivalent value']}
	demo_func(-1, -1, -1)
	#ValueError: {'b': ['must be greater than 1']}
	demo_func(1, 2, 'abc')
	#ValueError: {'c': ['must be an instance of int or its subclasses']}
```

总体上，整个程序都简洁了很多。
现在ArgsValidator装饰器只需要接收一个校验规则的字典。字典的键是异常类型；字典的值还是一个字典，它是validator模块的validate方法用来检验参数所使用的规则字典。
我们在内嵌的装饰函数中，首先还是和之前一样获取被装饰函数的所有参数并转换成有序关键字参数字典形式；然后遍历所有的规则并调用validate方法进行校验，校验失败则抛出对应的异常，所有的规则都校验成功则调用被装饰函数执行真正的业务处理逻辑。

### 持续改进
至此我们完美解决了所有的问题。
真的吗？还不是呢。
1. 这个装饰器类本身的参数校验呢？他的那个规则字典是一个嵌套的字典，这是一个相对复杂的数据结构，在创建，传递和使用的时候都容易发生错误。一个用于校验参数的切面功能居然没有校验自己的参数，未免太黑色幽默了。
2. 因为这个装饰器所支持的参数校验逻辑是基于validator.py，如果有一些复杂的校验逻辑是validator.py不支持的，那么应该怎么办呢？
3. 当前这个装饰器在校验出非法参数的时候，会提示哪个参数出现哪种校验失败。但是在一个庞大的工程里面，函数数量众多，不同函数的同名参数也数不胜数。这种情况下，当前的处理逻辑是不利于定位故障的。那有没有更友好的方式协助使用者定位出错的具体位置呢？

上面的问题我都解决了，这里就列出大体的思路，具体的实现就不说了。
1. 同样使用validator.py，甚至最基本的if-else判断逻辑来确保：输入的规则是一个字典；字典的键是异常类型；值也是一个字典；值字典的键是一个字符串，值字典的值是一个列表；值字典的值列表里面的元素是validator.py中定义的Validator类型对象(这里不完全准确，主要是validator.py有一个奇怪的东东叫Required，说起来又太长篇大论了，就先忽略了。)
2. 可以通过集成validator.py中定义的Validtor类，定义合适的属性，在其中的__call__方法中实现具体的校验判断逻辑。
3. 使用function.\_\_code__属性下面的各种属性获取，出错的被装饰函数的名字，位置，所在的源文件信息，快速定位故障点。

## 参考
[菜鸟教程-装饰器](https://www.runoob.com/w3cnote/python-func-decorators.html)
[Pyhton参数校验的进化](https://yq.aliyun.com/articles/682443)
[validator.py](https://validatorpy.readthedocs.io/en/latest/index.html)