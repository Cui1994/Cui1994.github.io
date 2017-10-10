---
layout: post
category: "study"
title:  "大话设计模式Python实现"
tags: [设计模式]
---

学习自大话设计模式，将其C#语言实现翻译为Python，同时参照此系列博客：[Python设计模式][1] 加深理解。

## 1.简单工厂模式

面向对象目标：

+ 可维护：只需要改动需求制定功能的类或模块
+ 可复用：可以用在不同的环境下
+ 可扩展：增加功能只需要增加相应的模块
+ 灵活性好：可以任意组合功能

耦合性：更改一个功能不需要接触其他功能（即使功能都是相似的），考虑通过封装、继承和多态将耦合性降低。

根据需求实例化要实例的对象（这些对象可能均继承自同一基类）。

缺点：违反了高内聚职责分配原则 [职责分配原则][2]

计算器实例：

```python
# -*- coding:utf-8 -*-
import re=

class Operation(object):
	def __init__(self, a, b):
		self.a = a
		self.b = b

	def calculate_result(self):
		result = 0
		return result

class AddOperation(Operation):
	def calculate_result(self):
		result = self.a + self.b
		return result

class SubtractOperation(Operation):
	def calculate_result(self):
		result = self.a - self.b
		return result

class MultiplicationOperation(Operation):
	def calculate_result(self):
		result = self.a * self.b
		return result

class DivisionOperation(Operation):
	def calculate_result(self):
		result = self.a / self.b
		return result

class OperationFactory(object):
	def create_operation(self, operation, a, b):
		if operation == '+':
			return AddOperation(a, b)
		elif operation == '-':
			return SubtractOperation(a, b)
		elif operation == '*':
			return MultiplicationOperation(a, b)
		elif operation == '/':
			return DivisionOperation(a, b)
		else:
			raise ValueError

if __name__ == '__main__':
	input_string = raw_input('Enter your operation such as \'10+11=\':')
	pattern = re.compile(r'(\d+)(\+|\-|\*|\/)(\d+)')
	items = pattern.findall(input_string)
	try:
		a = int(items[0][0])
		operation = items[0][3]
		b = int(items[0][2])
		print OperationFactory().create_operation(operation, a, b).calculate_result()
	except:
		print 'Wrong Input.'
```

## 2.策略模式

将一系列算法家族封装，算法家族完成的是同一类功能的不同实现（都是用来解决同一个问题的，只是不同情况需要应用不同的算法），降低了客户端和算法类之间的耦合。

缺点：增加策略时仍然需要到`Context`类中增加一个新的判断分支。

超市活动实例：

```python
# -*- coding:utf-8 -*-

class Strategy(object):
	# 抽象算法类
	def algorithm_interface(self):
		raise NotImplementedError()

class NormalStrategy(Strategy):

	def algorithm_interface(self, money):
		return money

class RebateStrategy(Strategy):

	def __init__(self, rebate):
		self.rebate = rebate

	def algorithm_interface(self, money):
		return money*self.rebate

class ReturnStrategy(Strategy):

	def __init__(self, each_money, return_money):
		self.each_money = each_money
		self.return_money = return_money

	def algorithm_interface(self, money):
		return money-(money/self.each_money)*self.return_money

class Context(object):
	# 上下文，封装策略的实现细节
	def __init__(self, strategy_type):
		if strategy_type == 'normal':
			self.strategy = NormalStrategy()
		elif strategy_type == '0.8 rebate':
			self.strategy = RebateStrategy(0.8)
		elif strategy_type == '300 return 100':
			self.strategy = ReturnStrategy(300, 100)

	def context_interface(self, money):
		return self.strategy.algorithm_interface(money)

if __name__ == '__main__':
	money = int(raw_input('Enter money:'))
	strategy_type = raw_input('Enter strategy type:')
	print Context(strategy_type).context_interface(money)
```

## 3.单一职责原则

类的功能要尽量单一——利于解耦

## 4.开放封闭原则

最扩展开放，最修改封闭。（写好的类尽量不要去改动他）
面对需求，对程序的改动是通过增加新代码进行的，而不是更改现有的代码。
在最开始写代码时，假定变化不会发生，当变化发生之后，就要创建抽象来隔离变化。

## 5.依赖倒转原则

抽象不应该依赖于细节，细节应该依赖于抽象。
要对接口编程，而不要对实现编程。
里氏代换原则：子类必须能够替换掉他们的父类型。

## 6.装饰模式

把每个需要装饰的功能放在单独的类中，需要使用新功能时只需要用装饰的类去包装原有的核心类，将类的核心职责和装饰功能分开，避免增加核心类的复杂度。

python自带的装饰器也是一种装饰模式的实现。

穿衣服实例

```python
# -*- coding:utf-8 -*-
class Person(object):

	def __init__(self):
		self.name = ''

	def set_name(self, name):
		self.name = name

	def show(self):
		print u'装扮的{0}'.format(self.name)

class Finery(Person):

	def set_decorate(self, component):
		self.component = component

	def show(self):
		if self.component:
			self.component.show()

class TShirts(Finery):

	def show(self):
		print u'T-shirt.'
		super(TShirts, self).show()

class BigTrouser(Finery):

	def show(self):
		print u'Big-Trouser.'
		super(BigTrouser, self).show()

if __name__ == '__main__':
	xiao_ming = Person()
	xiao_ming.set_name('xiaoMing')

	print u'第一种装扮：'
	t_shirt = TShirts()
	big = BigTrouser()

	t_shirt.set_decorate(xiao_ming)
	big.set_decorate(t_shirt)

	big.show()
```

## 7.代理模式

用代理对象去调用真是对象的接口，可以完成一些额外的事。

代理和真是对象公用一个接口（继承自同一基类）。代理此接口的真正目的是调用真实对象的接口。

代码实例：

```python
# -*- coding:utf-8 -*-

class Subject(object):

	def request(self):
		raise NotImplementedError

class RealSubject(Subject):

	def request(self):
		print 'Real Request.'

class ProxySubject(Subject):

	def __init__(self):
		self.real = RealSubject()

	def request(self):
		self.real.request()

if __name__ == '__main__':
	proxy = ProxySubject()
	proxy.request()
```

## 8.工厂方法模式

将简单工厂模式实例化类的时机延后到工厂子类进行，克服了简单工厂模式违背封闭原则的缺点。

但其仍有缺点：判断分支从工厂中转移到了客户端中进行。相当于绕了一圈又绕回来了。

计算器实例：

```python
# -*- coding:utf-8 -*-
import re

class Operation(object):
	def __init__(self, a, b):
		self.a = a
		self.b = b

	def calculate_result(self):
		result = 0
		return result

class AddOperation(Operation):
	def calculate_result(self):
		result = self.a + self.b
		return result

class SubtractOperation(Operation):
	def calculate_result(self):
		result = self.a - self.b
		return result

class MultiplicationOperation(Operation):
	def calculate_result(self):
		result = self.a * self.b
		return result

class DivisionOperation(Operation):
	def calculate_result(self):
		result = self.a // self.b
		return result

class IFactory(object):
	def create_operation(self, a, b):
		raise NotImplementedError

class AddFactory(IFactory):
	def create_operation(self, a, b):
		return AddOperation(a, b)

class SubFactory(IFactory):
	def create_operation(self, a, b):
		return SubtractOperation(a, b)

class MulFactory(IFactory):
	def create_operation(self, a, b):
		return MultiplicationOperation(a, b)

class DivFactory(IFactory):
	def create_operation(self, a, b):
		return DivisionOperation(a, b)

if __name__ == '__main__':
	input_string = raw_input('Enter your operation such as \'10+11=\':')
	pattern = re.compile(r'(\d+)(\+|\-|\*|\/)(\d+)')
	items = pattern.findall(input_string)
	try:
		a = int(items[0][0])
		operation = items[0][1]
		b = int(items[0][2])
		if operation == '+':
			oper = AddFactory().create_operation(a, b)
		elif operation == '-':
			oper = SubFactory().create_operation(a, b)
		elif operation == '*':
			oper = MulFactory().create_operation(a, b)
		elif operation == '/':
			oper = DivFactory().create_operation(a, b)
		print oper.calculate_result()
	except:
		print 'Wrong Input.'
```

## 9.原型模式

从一个对象再创建一个可定制的对象，而且不需要知道任何创建的细节。（不需要再手动实例化一个新实例）
tip：这里涉及了深浅拷贝的概念。

优点：隐藏创建细节，提高性能。（不需要每次都调用构造函数）

```python
import copy

class Book(object):
	def __init__(self, name, authors, price):
		self.name = name
		self.authors = authors
		self.price = price

	def clone(self, **kwargs):
		book_copy = copy.deepcopy(self)
		book_copy.__dict__.update(**kwargs)
		return book_copy

book1 = Book('Python', ['Tom', 'Jack'], 10)
book2 = book1.clone(price=20)
print book2.__dict__
# {'price': 20, 'name': 'Python', 'authors': ['Tom', 'Jack']}
```

## 10. 模板方法模式

比较常见的设计模式，制定一种工作流或算法的特定骨架而将具体实现放到子类中。

```python
class FatherClass(object):
    def tempmethod(self):
        print "first step: red"
        self.childmethod()
        print "third step: blue"
        
    def childmethod(self):
        raise NotImplementedError
    
class ChildClass1(FatherClass):
    def childmethod(self):
        print "second step: green"
        
class ChildClass2(FatherClass):
    def childmethod(self):
        print "second step: yellow"

```

## 11.迪米特法则

如果两个类不必彼此直接通信，那么这两个类就不应当发生直接的相互作用。如果一个类要调用另一个类的某一个方法，可以通过第三者转发这个调用。

强调类之间的松耦合。耦合度越低，越利于复用。

## 12.外观模式

给一组系统方法一个统一的接口，提供了一个更高层的接口方法。

有点类似代理模式。

```python
class ModuleOne(object):
    def Create(self):
        print 'create module one instance'

    def Delete(self):
        print 'delete module one instance'

class ModuleTwo(object):
    def Create(self):
        print 'create module two instance'

    def Delete(self):
        print 'delete module two instance'

class Facade(object):
    def __init__(self):
        self.module_one = ModuleOne()
        self.module_two = ModuleTwo()

    def create_module_one(self):
        self.module_one.Create()

    def create_module_two(self):
        self.module_two.Create()

    def create_both(self):
        self.module_one.Create()
        self.module_two.Create()

    def delete_module_one(self):
        self.module_one.Delete()

    def delete_module_two(self):
        self.module_two.Delete()

    def delete_both(self):
        self.module_one.Delete()
        self.module_two.Delete()
```

## 13.建造者模式

讲一个复杂对象的构建和他的表示分离，这样可以用同样的构建方式创建不同的表示。

优点：对象内部的构建顺序是稳定的，建造者隐藏了产品如何组装。

```python
# -*- coding:utf-8 -*-

class Builder(object):
# 抽象建造者类，也可以说是产品类
	def part1(self):
		raise NotImplementedError

	def part2(self):
		raise NotImplementedError

class Builder1(object):

	def part1(self):
		print 'builder1 part1'

	def part2(self):
		print 'builder1 part2'

class Builder2(object):

	def part1(self):
		print 'builder2 part1'

	def part2(self):
		print 'builder2 part2'

class Director(object):

	def build(self, builder):
		builder.part1()
		builder.part2()

if __name__ == '__main__':
	builder = Builder1()
	director = Director()

	director.build(builder)
```

也可以将`Director`的代码已到`Builder`类中，实现模板方法模式。

```python
class Builder(object):
    # ···
    def build(self):
        self.part1()
        self.part2()
```

## 14.观察者模式

又称作Pub/Sub模式
定义了一种一对多的依赖关系，让众多观察者可以同时关注某一主题，当主题发生变化时每个观察者都会更新自己的状态。
许多MQ都是通过这一模式实现的。


```python
class Topic(object):

	def __init__(self):
		self.obs = []

	def attach(self, observer):
		self.obs.append(observer)

	def detach(self, observer):
		self.obj.remove(observer)

	def notify(self, message):
		for observer in self.obs:
			observer.update(message)

class Observer(object):

	def update(self, message):
		raise NotImplementedError

class ConcreteObserver(Observer):

	def update(self, message):
		print message

if __name__ == '__main__':
	topic = Topic()
	ob1 = ConcreteObserver()
	ob2 = ConcreteObserver()
	topic.attach(ob1)
	topic.attach(ob2)
	topic.notify("hello")

```

## 15.抽象工厂模式

创建具有一定功能的产品实现时，需要先创建具体的工厂类，再由工厂类创建具有特定实现的产品对象。


```python
# -*- coding:utf-8 -*-

class IUser(object):

	def insert_user(self, user):
		raise NotImplementedError

	def get_user(self, userid):
		raise NotImplementedError

class IRole(object):
	def insert_role(self, role):
		raise NotImplementedError

	def get_role(self, roleid):
		raise NotImplementedError

class SqlServerUser(IUser):

	def insert_user(self, user):
		print u"在SQL Server中给User表增加一条记录."

	def get_user(self, userid):
		print u"在SQL Server中拿到一条uer信息。"

class SqlServerRole(IRole):
	def insert_role(self, role):
		print u"在SQL Server中给Role表增加一条记录"

	def get_role(self, roleid):
		print u"在SQL Server中拿到一条role记录"

# 产品类        
class AccessUser(IUser):
	def insert_user(self, user):
		print u"在Access中给User表增加一条记录."

	def get_user(self, userid):
		print u"在Access中拿到一条uer信息。"

class AccessRole(IUser):
	def insert_role(self, role):
		print u"在Access中给Role表增加一条记录"

	def get_role(self, roleid):
		print u"在Access中拿到一条role记录"

# 抽象工厂类
class IFactory(object):
	def create_user(self):
		raise NotImplementedError

	def create_role(self):
		raise NotImplementedError

# 具体工厂类
class SqlServerFactory(IFactory):

	def create_user(self):
		return SqlServerUser()

	def create_role(self):
		return SqlServerRole()

# 具体工厂类
class AccessFactory(IFactory):

	def create_user(self):
		return AccessUser()

	def create_role(self):
		return AccessRole()

class User(object):

	def __init__(self, id):
		self.id = id

class Role(object):

	def __init(self, id):
		self.id = id


if __name__ == '__main__':

	user = User(1)
	factory = SqlServerFactory()
	iu = factory.create_user()

	iu.insert_user(user)
	iu.get_user(1)
```

也可以考虑用引入简单工厂加反射进行优化。

## 16.状态模式

对象的行为取决于它的状态，并且他需要在运行时刻根据状态改变它的行为。

此时会产生大量判断语句，使用状态模式可以消除这些判断语句，降低耦合性。

工作实例：

```python
# -*- coding:utf-8 -*-

class State(object):
	def write_program(work):
		raise NotImplementedError

class ForenoonState(State):
	def write_program(self, wrok):
		if work.hour < 12:
			print u"当前时间: {}点， 上午工作，精神百倍".format(work.hour)
		else:
			wrok.set_state(NoonState())
			work.write_program()

class NoonState(State):
	def write_program(self, work):
		if work.hour < 13:
			print u"当前时间：{}点， 饿了，午饭； 困了，午休".format(work.hour)
		else:
			work.set_state(AfternoonState())
			work.write_program()

class AfternoonState(State):
	def write_program(self, work):
		if work.task_finished:
			work.set_state(RestState())
		if work.hour < 17:
			print u"当前时间：{}点， 下午状态不错，继续努力".format(work.hour)
		else:
			work.set_state(EveningState())
			work.write_program()

class EveningState(State):
	def write_program(self, work):
		print u"当前时间： {}点， 又要加班了！".format(work.hour)

class RestState(State):
	def write_program(self, work):
		print u"当前时间： {}点， 下班回家啦".format(work.hour)

class Work(object):
	def __init__(self):
		self.current = ForenoonState()
		self.hour = 9
		self.task_finished = False

	def set_hour(self, hour):
		self.hour = hour

	def set_finished(self, finished):
		self.set_finished = finished

	def set_state(self, state):
		self.current = state

	def write_program(self):
		self.current.write_program(self)

if __name__ == "__main__":
	work = Work()
	work.set_hour(9)
	work.write_program()
```

## 17.适配器模式

当一个类的功能和数据相同而接口不同时，需要适配器模式充当翻译角色。

缺点：亡羊补牢

NBA翻译实例：

```python
# -*- coding:utf-8 -*-

class Player(object):
	def __init__(self, name):
		self.name = name

	def attack(self):
		print u"{} 进攻".format(self.name)

	def defense(self):
		print u"{} 防守".format(self.name)

class ForeignPlayer(object):

	def __init__(self, name):
		self.name = name

	def jingong(self):
		print u"{} 进攻".format(self.name)

	def fangshou(self):
		print u"{} 防守".format(self.name)

class Translator(Player):

	def __init__(self, name):
		super(Translator, self).__init__(name)
		self.fp = ForeignPlayer(self.name)

	def attack(self):
		self.fp.jingong()

	def defense(self):
		self.fp.fangshou()

if __name__ == '__main__':
	b = Player("Bdier")
	b.attack()

	m = Player("Mical")
	m.attack()

	ym = Translator("YaoMing")
	ym.attack()
	ym.defense()
```






  [1]: http://www.jianshu.com/p/84ae207ccaf7
  [2]: http://www.cnblogs.com/sevenyuan/archive/2010/03/05/1678730.html
  [3]: http://www.jianshu.com/p/84ae207ccaf
