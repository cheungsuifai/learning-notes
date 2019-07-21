
# Python编码规范
## 前言
这是一个实践性的使用说明，不会涉及分析深入底层技术。
## 标准
Python中有一个概念叫PEP（Python Enhancement Proposals）。PEP是Python的增强建议书，为Python社区提供各种增强功能的技术规格，也是提交新特性，以便让社区指出问题，精确化技术文档。
PEP可以分为三大类：
* **信息类**
信息类PEP提供信息，包括告知类信息，指导类信息等。如PEP 20（The Zen of Python，即著名的Python之禅）PEP 404 (Python 2.8 Un-release Schedule，即宣告不会有Python2.8版本)。
* **流程类**
流程类PEP主要是Python本身之外的周边信息。例如PEP 1（PEP Purpose and Guidelines，即关于PEP的指南）、PEP 347（Migrating the Python CVS to Subversion，即关于迁移Python代码仓）。
* **标准类**
标准类PEP主要描述了Python的新功能和新实践（implementation），是数量最多的提案。例如PEP 498（Literal String Interpolation，字面字符串插值）。

而关于编码规范则在编号为8的PEP中。称为PEP8。它针对Python的编码作出了详细的规定，例如：

* **缩进**
使用4个空格缩进
绝对不要混用 tab 和空格
* **空行**
function 和 class 顶上两个空行
class 的 method 之间一个空行
函数内逻辑无关的段落之间空一行，不要过度使用空行
不要把多个语句写在一行，然后用 ; 隔开
if/for/while 语句中，即使执行语句只有一句，也要另起一行
* **换行**
每一行代码控制在 80 字符以内 （这规定太傻叉了）
使用 \ 或 () 控制换行


## 检查
现在标准是有了，那该如何执行呢？
纯粹人工按照规定修正代码是很低效的，既然是程序员就应该用程序解决检查的问题。
其实这方面的工具还有挺多的，例如: pep8(好像现在叫pycodestyle), pylint.
这里只介绍flake8，而且是把它作为外部工具集成在Pycharm IDE中实现编码规范检查。

### 安装Pycharm
要啥自行车，官网下载自行安装。

### 安装flake8
要啥自行车，pip或者Anaconda自行安装。
pip:
```shell
pip install flake8
```
Anaconda:
```shell
conda install flake8
```

### 配置
1. 启动Pycharm
2. 选择**File->Settings->Tools->External Tools**
3. 点击右面板左上角的绿色+号
4. 在配置对话框中填入下列信息 
 
| 选项 | 取值 |
| --- | --- |
| Name | flake8 |
| Program | \$PyInterpreterDirectory$/python |
| Arguments | -m flake8 --show-source --statistics \$ProjectFileDir$ --max-line-length 119 |
| Working directory |  \$ProjectFileDir$ |

 注：
 1. 我的环境中Program选项使用宏不能识别，我没有纠结，直接使用了本地的Python目录：**C:\Program Files\Python36\python.exe**
 2. PEP8规定那个每行宽度不能超过80的规定太死板了，我把它的上限调成了120，所以加多了一个参数：
 **--max-line-length 119**。毕竟PEP8一开篇就说：A Foolish Consistency is the Hobgoblin of Little Minds.

### 执行
在配置号Flake8的External Tool后，就可以使用了。通过右键然后在菜单中选择External Tools -> Flake8就可以执行了。我还为它设置了快捷键Ctrl+;，用起来简直不要太爽。

## 修正
在第一次使用Flake8检查的刚刚写完一堆随心所欲的代码时，你会发现有你的代码在PEP8的规范下，简直就是
一个神奇的工具autopep8。

如果纯手工修正编码风格的问题将会很好费时间，幸好程序员都很懒，发明了各种工具来偷懒。

### IDE
Pycharm IDE提供了代码规整功能，使用Pycharm IDE的同学可以直接使用快捷键 Ctrl+Alt+L自动按照PEP8规范调整代码，但这个功能也并不能修复所有的问题。

### autopep8
另外有一个神奇的工具也能自动根据PEP8调整代码格式：autopep8.
#### 安装方法
要啥自行车
```shell
pip install autopep8
```
#### 使用方法
使用方法如下所示，--in-place表示直接在原文件修改，--aggressive表示使用激进模式。更多的参数可以通过 --help查看。
```shell
$ autopep8 --in-place --aggressive --aggressive <filename>
```

#### 集成方法
Pycharm怎么可能不支持autopep8的集成呢？而且聪明的同学肯定就猜到还是用External Tools的配置来集成的。

配置如下：
| 选项 | 取值 |
| --- | --- |
| Name | autopep8 |
| Program | \$PyInterpreterDirectory\$autopep8 |
| Arguments | --in-place --aggressive --aggressive \$FilePath\$ |
| Working directory |  \$ProjectFileDir\$ |
|Output filters：| \$FILE_PATH\$\\:\$LINE\$\\:\$COLUMN\$\\:.\*|

注：
这样配置之后，需要在选定python的源文件后再执行。如果选定了目录来执行，会提示权限不足（Permission Denied）。

## 结语
基本就这样，其实内容不多，但是比较零散，都整理一遍也耗费了一定的时间。希望大家都能写出规范的代码。

## 参考
1. [PEP8](https://www.python.org/dev/peps/pep-0008/)
2. [flake8](https://flake8.readthedocs.io/en/latest/)
3. [autopep8](https://pypi.org/project/autopep8/)