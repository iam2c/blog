# 0x01 写在前面
本文主要介绍：
1. Generator的简单用法。
2. Generator的底层实现。

如果你比较了解Generator的用法，仅想了解底层实现，可以直接跳到底层实现部分。

本文分析的PHP源码版本为：7.0.29。

> 此文为个人的学习笔记，意在对自己的学习过程进行总结。由于个人能力有限，错漏在所难免，欢迎批评指正。

# 0x02 Generator的用法
Generator，中文翻译：生成器，是PHP 5.5开始支持的语法。

## 2.1 [生成器总览](http://php.net/manual/zh/language.generators.overview.php)

> 生成器提供了一种更容易的方法来实现简单的对象迭代，相比较定义类实现 Iterator 接口的方式，性能开销和复杂性大大降低。

> 生成器允许你在 foreach 代码块中写代码来迭代一组数据而不需要在内存中创建一个数组, 那会使你的内存达到上限，或者会占据可观的处理时间。相反，你可以写一个生成器函数，就像一个普通的自定义函数一样, 和普通函数只返回一次不同的是, 生成器可以根据需要 yield 多次，以便生成需要迭代的值。

## 2.2 生成器对象
> 当一个生成器的函数的被调用时，对返回内置类Generator的一个实例化对象。这个对象实现了Iterator接口，跟迭代器一样可以向前迭代，并且提供了维护这个对象的状态的接口，包括向它发送值和从它接收值。

## 2.3 [生成器语法](http://php.net/manual/zh/language.generators.syntax.php)
> 一个生成器函数看起来像一个普通的函数，不同的是普通函数返回一个值，而一个生成器可以yield生成许多它所需要的值。

> 当一个生成器被调用的时候，它返回一个可以被遍历的对象.当你遍历这个对象的时候(例如通过一个foreach循环)，PHP 将会在每次需要值的时候调用生成器函数，并在产生一个值之后保存生成器的状态，这样它就可以在需要产生下一个值的时候恢复调用状态。

> 一旦不再需要产生更多的值，生成器函数可以简单退出，而调用生成器的代码还可以继续执行，就像一个数组已经被遍历完了

> PHP 5是不可以有返回值的，如果这样做会导致编译错误。但是一个空的return语句是可以的，这会终止生成器的执行。PHP 7支持返回值，使用Generator::getReturn()获取返回值。

### 2.3.1 yield关键字
> 生成器函数的核心是yield关键字。它最简单的调用形式看起来像一个return申明，不同之处在于普通return会返回值并终止函数的执行，而yield会返回一个值给循环调用此生成器的代码并且只是暂停执行生成器函数。 


理论显得空洞无力，show me your code，那就来看一段简单的代码，以便更容易理解生成器语法：

代码片段2.1：
```
<?php
function gen_one_to_three() {
    for ($i = 1; $i <= 3; $i++) {
        yield $i;
    }
}

$generator = gen_one_to_three();
foreach ($generator as $value) {
    echo "$value\n";
}
```

![img](https://raw.githubusercontent.com/iam2c/blog/master/assets/generator/1.gif)

说明：
1. 执行`$generator = gen_one_to_three();`，这时不会执行生成器函数gen_one_to_three()里面的代码，而是返回一个Generator对象，也就是说$generator是一个Generator对象。
2. `foreach ($generator as $value)`遍历Generator对象，因为Generator实现了Iterator接口，可以用foreach进行迭代。这时就会调用生成器函数gen_one_to_three()，于是执行gen_one_to_three()的代码。
3. 因为是首次调用，所以从开始执行，执行for循环，此时$i=1，执行到`yield $i;`相当于生成了一个值1，并且保存了当前的状态（比如$i=1、执行`yield $i;`这里）并暂停执行。
4. foreach获取到这个值1，并echo输出。
5. 继续遍历foreach，这是会调用生成器函数，并恢复从上次保存的状态（包括变量值，和执行到的位置）继续执行，`$i++`，这是$i=2。
6. for循环继续执行，再次执行`yield $i;`相当于生成一个值2，并且保存了当前的状态并暂停执行。
7. foreach获取到这个值2，并echo输出。
8. foreach继续执行，继续调用生成器函数，这是`$i++`，$i=3，执行`yield $i;`生成一个值3给$value并输出$value。
9. foreach继续执行，但是生成器函数没有生成值了（valid()返回false，看后面的`代码片段2`），所以结束foreach遍历。

### 2.3.2 yield from 
> PHP 7允许您使用yield from关键字从另一个生成器、Traversable对象或数组中生成值（后面简称**委托对象**），这叫生成器委托。 生成器将从内嵌生成器、对象或数组中生成所有值，直到它不再有效，然后继续生成器的执行。

代码片段2.3.2：
```php
<?php
function count_to_ten() {
    yield 1;
    yield 2;
    yield from [3, 4];
    yield from new ArrayIterator([5, 6]);
    yield from seven_eight();
    yield 9;
    yield 10;
}

function seven_eight() {
    yield 7;
    yield from eight();
}

function eight() {
    yield 8;
}

foreach (count_to_ten() as $num) {
    echo "$num ";
}
?> 
上例会输出：

1 2 3 4 5 6 7 8 9 10 
```

![img](https://raw.githubusercontent.com/iam2c/blog/master/assets/generator/2-1.gif)

以上的引用内容来自于[PHP帮助手册](http://php.net/manual/zh/language.generators.php)，例子也基本来自手册，我只是加了一些说明，以便帮助更好的理解其语法。

## 2.4 Generator类
前面说Generator类实现了Iterator接口，那到底有哪些成员方法呢？
```php
Generator implements Iterator {
    public mixed current ( void )
    public mixed key ( void )
    public void next ( void )
    public void rewind ( void )
    public mixed send ( mixed $value )
    public void throw ( Exception $exception )
    public bool valid ( void )
    public void __wakeup ( void )
}
```

Generator比起Iterator接口，增加了`send()`、`throw()`以及`__wakeup()`方法。

既然实现了Iterator接口，那上面的**代码片段2.3.1**也可以改成下面的，执行结果一样的：

代码片段2.4.1：
```php
<?php
function gen_one_to_three() {
    for ($i = 1; $i <= 3; $i++) {
        yield $i;
    }
}

$generator = gen_one_to_three();
while ($generator->valid()) {
    echo "{$generator->current()}\n";
    $generator->next();
}
```

![img](https://raw.githubusercontent.com/iam2c/blog/master/assets/generator/2.gif)

## 2.5 Generator方法

### 2.5.1 Generator::__wakeup()
这是一个[魔术方法](http://php.net/manual/zh/language.oop5.magic.php#object.wakeup)，当一个对象被反序列化时会调用，但生成器对象不能被序列化和反序列化，所以`__wakeup()`方法抛出一个异常以表示生成器不能被序列化。

### 2.5.2 Generator::send()
前面生成器对象部分提到：可以从生成器对象接收值和向它发送值。yield就是从它接收值，那发送值是什么呢？就是这个send()方法。

[PHP帮助文档的介绍](http://php.net/manual/zh/generator.send.php)：
```php
public mixed Generator::send ( mixed $value )
```
> 向生成器中传入一个值，并且当做 yield 表达式的结果，然后继续执行生成器。 

> 如果当这个方法被调用时，生成器不在 yield表达式，那么在传入值之前，它会先运行到第一个 yield 表达式.

先来理解第一段话：
> 向生成器中传入一个值，并且当做 yield 表达式的结果，然后继续执行生成器。 

yield后生成了值，还可以用这个生成器对象的send()方法发送一个值，而这个值作为表达式的结果，然后拿生成器函数里面可以获取这个值，然后**继续执行生成器**。看下面的代码：

代码片段2.5.1：

```php
<?php
function gen_one_to_three() {
    for ($i = 1; $i <= 3; $i++) {
        $cmd = (yield $i);
        if ($cmd === 'exit') {
            return;
        }
    }
}

$generator = gen_one_to_three();
foreach ($generator as $value) {
    echo "$value\n";
    $generator->send('exit');
}
```
![img](https://raw.githubusercontent.com/iam2c/blog/master/assets/generator/3.gif)

说明：
1. 执行`$generator = gen_one_to_three();`，这时不会执行生成器函数gen_one_to_three()里面的代码，而是返回一个Generator对象，也就是说$generator是一个Generator对象。
2. `foreach ($generator as $value)`遍历Generator对象，因为Generator实现了Iterator接口，可以用foreach进行迭代。这时就会调用生成器函数gen_one_to_three()，于是执行gen_one_to_three()的代码。
3. 因为是首次调用，所以从开始执行，执行for循环，此时$i=1，执行到`$cmd = (yield $i);`相当于生成了一个值1，并且保存了当前的状态（比如$i=1、执行`yield $i;`这里）并暂停执行。
4. foreach获取到这个值1，赋给$value，并echo输出。
5. 执行`$generator->send('exit');`向生成器函数里面发送值"exit"。
6. 生成器函数拿到这个值"exit"，作为`yield $i;`表达式的值，然后赋给$cmd，也就是`$cmd = (yield $i);`相当于`$cmd = "exit";`，继续**执行生成器函数**。
7. `if ($cmd === 'exit')`条件成立，所以执行return，终止生成器函数的运行。

接下来，看看第二段话：
> 如果当这个方法被调用时，生成器不在 yield表达式，那么在传入值之前，它会先运行到第一个 yield 表达式。

也就是说不一定用foreach来执行生成器函数，send()也可以，知道遇到第一个yield表达式，后面步骤就按照第一段话的步骤处理。

### 2.5.3 Generator::throw()

向生成器中抛入一个异常。

代码片段2.4：
```php
<?php
function gen_one_to_three() {
    for ($i = 1; $i <= 3; $i++) {
        yield $i;
    }
}

$generator = gen_one_to_three();
foreach ($generator as $value) {
    echo "$value\n";
    $generator->throw(new \Exception('test'));
}
```

说明：

1. 执行`$generator = gen_one_to_three();`，这时不会执行生成器函数gen_one_to_three()里面的代码，而是返回一个Generator对象，也就是说$generator是一个Generator对象。
2. `foreach ($generator as $value)`遍历Generator对象，因为Generator实现了Iterator接口，可以用foreach进行迭代。这时就会调用生成器函数gen_one_to_three()，于是执行gen_one_to_three()的代码。
3. 因为是首次调用，所以从开始执行，执行for循环，此时$i=1，执行到`yield $i;`相当于生成了一个值1，并且保存了当前的状态（比如$i=1、执行`yield $i;`这里）并暂停执行。
4. foreach获取到这个值1，并echo输出。
5. 执行`$generator->throw(new \Exception('test'));`，相当于在生成器函数`yield $i;`处抛出了一个异常`new \Exception('test')`。

这节只简单介绍了生成器类Generator的用法，如果想要实现更复杂的功能，比较推荐鸟哥的[《在PHP中使用协程实现多任务调度》](http://www.laruence.com/2015/05/28/3038.html)。

# 0x04 生成器的底层实现

从前面几节我们初步知道生成器函数跟别的函数不一样，别的函数在返回返回时，除了静态变量外其他的都会被销毁，下次进来还是新的状态，也就是不会保存状态值，但生成器函数每次yield是会保存状态，包括变量值和运行位置，下次调用时从上次运行的位置后面继续运行。了解Generator的运行机制，需要对Zend VM有一定了解，可以先阅读这篇文章[《Zend引擎执行流程》](https://github.com/pangudashu/php7-internal/blob/master/3/zend_executor.md)。

从PHP语法层面分析，底层实现应该具有：
1. Generator实现了迭代器接口
2. 生成器函数调用时返回Generator对象
3. yield后会保存函数的局部遍历和运行位置（内存不会被销毁）

下面，我们从源码分析Generator的底层实现。

**本节注意**：
1. 代码中`// ...`表示省略一部分代码。
2. 代码中会加一些注释说明，以便更好地了解代码。
3. Zend/xxx.c:767-864表示Zend目录下的xxx.c文件，行数为767至864行。


## 4.1 Generator类的注册及其存储结构
先从数据结构入手，类和对象底层的结构分别为：`zend_class_entry` 和`zend_object`。类产生在是编译时，而对象产生是在运行时。Generator是一个内置类，具有跟其他类共同的性质，但也有自己不同的特性。

本文不会介绍类和对象的内部实现，感兴趣的可以阅读[《面向对象实现-类》](https://github.com/pangudashu/php7-internal/blob/master/3/zend_class.md)和[《面向对象实现-对象》](https://github.com/pangudashu/php7-internal/blob/master/3/zend_object.md)。如果你对这些知识不太了解，请先阅读上面两篇文章，以便更好地理解后面的内容。

内置类在PHP模块初始化（MINIT）的时候就注册了。调用路径为：ZEND_MINIT_FUNCTION(core) -> zend_register_default_classes() -> zend_register_generator_ce()：

代码片段4.1.1：

```C
void zend_register_generator_ce(void) /* {{{ */
{
	zend_class_entry ce;

	INIT_CLASS_ENTRY(ce, "Generator", generator_functions); // 初始化Generator类，主要其方法
	zend_ce_generator = zend_register_internal_class(&ce);  // 注册为内部类
	zend_ce_generator->ce_flags |= ZEND_ACC_FINAL; // 设置为final类，表示不能被继承。
	/* 下面3个函数时钩子函数，内部类用到，用户自定义的会使用默认函数 */
	zend_ce_generator->create_object = zend_generator_create; // 创建对象
	zend_ce_generator->serialize = zend_class_serialize_deny; // 序列化，zend_class_serialize_deny表示不能序列化
	zend_ce_generator->unserialize = zend_class_unserialize_deny; // 反序列化，zend_class_unserialize_deny表示不能反序列化

	/* get_iterator has to be assigned *after* implementing the inferface */
	zend_class_implements(zend_ce_generator, 1, zend_ce_iterator); // 实现zend_ce_iterator类，也就是Iterator
	zend_ce_generator->get_iterator = zend_generator_get_iterator;  // 遍历方法，这也是个钩子方法，用户自定义的使用默认的
	zend_ce_generator->iterator_funcs.funcs = &zend_generator_iterator_functions; // 遍历相关的方法（valid/next/current等）使用自己的

    /* 下面几个是对象（Generator类的实例）相关的 */
	memcpy(&zend_generator_handlers, zend_get_std_object_handlers(), sizeof(zend_object_handlers)); // 先使用默认的，后面的相应覆盖
	zend_generator_handlers.free_obj = zend_generator_free_storage; // 释放
	zend_generator_handlers.dtor_obj = zend_generator_dtor_storage; // 销毁
	zend_generator_handlers.get_gc = zend_generator_get_gc; // 垃圾回收相关
	zend_generator_handlers.clone_obj = NULL; // 克隆。禁止克隆
	zend_generator_handlers.get_constructor = zend_generator_get_constructor; // 构造

	INIT_CLASS_ENTRY(ce, "ClosedGeneratorException", NULL);
	zend_ce_ClosedGeneratorException = zend_register_internal_class_ex(&ce, zend_ce_exception);
}
```

从**代码片段4.1.1**可以看出：
1. Generator类实现了Iterator接口，但有些方法和Iterator默认的方法不太一样。比如不能序列化/反序列化、遍历方法（getIterator）不一样等。
2. Generator类不能被继承。
3. Generator类的实例不能被克隆等。

## 4.2 zend_generator结构体
在介绍后面的内容之前，我觉得有必要先了解zend_generator这个结构体，因为底层代码基本都是围绕着这个结构体来开展的。

代码片段4.2.1：

```C
typedef struct _zend_generator zend_generator;
struct _zend_generator {
	zend_object std;
	zend_object_iterator *iterator;
	/* 生成器函数的execute_data */
	zend_execute_data *execute_data;
	/* VM stack */
	zend_vm_stack stack;
	/* 当前元素的值 */
	zval value;
	/* 当前元素的键 */
	zval key;
	/* 返回值 */
	zval retval;
	/* 用来保存send()的值 */
	zval *send_target;
	/* 当前使用的最大自增key */
	zend_long largest_used_integer_key;
	/* yield from才用到，数组和非生成器的Traversables类用到，后面会介绍 */
	zval values;
	/* Node of waiting generators when multiple "yield *" expressions are nested. */
	zend_generator_node node;
	/* Fake execute_data for stacktraces */
	zend_execute_data execute_fake;
	/* 标识 */
	zend_uchar flags;
};
```
重点介绍几个重要的：
*  execute_data：生成器函数的上下文execute_data，包括当前运行到的位置、变量等状态信息，底层EX宏就是访问这个结构的成员。如果这个为NULL，则表明该生成器已经结束，也就是没有更多的值生成了。当生成器函数return时（没有显式return底层默认return NULL），execute_data变为NULL，后面会介绍。
*  vm_stack：VM栈，这个会在[Generator对象的创建](#Generator对象的创建)中详细介绍。
*  key：当前元素的key，每次yield都会更新此值，如果yield没有指定key（也就是yield $key => $value形式），则使用largest_used_integer_key值。
*  value：当前元素的value，也就是生成的值，每次yield都会更新此值。
*  retval：生成器的返回值，也就是return返回的值，可以通过Generator::getReturn()获取。
*  largest_used_integer_key：存储当前已使用的自增key，yield没有指定key时使用下一个自增值。
*  send_target：send()的值就存放在这里。
*  values：yield from委托对象时用到；yield from生成器不会存储在这里，使用后面的node存储关系。
*  node：存储生成器与其委托对象的关系，这个数据结构有点复杂，暂时不做介绍。

## 4.3 Generator对象的创建
从生成器语法可以看出，生成器函数（方法）具有：
1. 必须是个函数
2. 函数有yield关键字
3. 调用生成器函数返回Generator对象

### 4.3.1 编译阶段
先从编译PHP代码开始分析，PHP7会先把PHP代码编译成AST（Abstract Syntax Tree，抽象语法生成树），然后再生成opcode数组，每条opcode就是一条指令，每条指令都有相应的处理函数（handler）。这里面细讲起来篇幅很长，建议阅读[《PHP代码的编译》](https://github.com/pangudashu/php7-internal/blob/master/3/zend_compile.md)、[《词法解析、语法解析》](https://github.com/pangudashu/php7-internal/blob/master/3/zend_compile_parse.md)和[《抽象语法树编译流程》](https://github.com/pangudashu/php7-internal/blob/master/3/zend_compile_opcode.md)这几篇文章。

先来看第一个特征：必须是个函数。函数的编译，比较复杂，不是本文的重点，需要了解可以阅读[《函数实现》](https://github.com/pangudashu/php7-internal/blob/master/3/function_implement.md)。函数的开始先标识`CG(active_op_array)`，展开是`compiler_globals.active_op_array`，这是一个`zend_op_array`结构，在PHP中，每一个也就是独立的代码段（函数/方法/全局代码段）都会编译成一个`zend_op_array`，生成的opcode数组就存在`zend_op_array.opcodes`。

再来看第二个特征：函数有yield关键字。在词法语法分析阶段，如果遇到函数里面的表达式有yield，则会标识为生成器函数。看词法语法过程，在Zend/zend_language_parser.y:855：

代码片段4.3.1：

```txt
expr_without_variable:
		T_LIST '(' assignment_list ')' '=' expr
			{ $$ = zend_ast_create(ZEND_AST_ASSIGN, $3, $6); }
	|	variable '=' expr
			{ $$ = zend_ast_create(ZEND_AST_ASSIGN, $1, $3); }		
// ...

	|	T_YIELD { $$ = zend_ast_create(ZEND_AST_YIELD, NULL, NULL); } // 958行 
	|	T_YIELD expr { $$ = zend_ast_create(ZEND_AST_YIELD, $2, NULL); }
	|	T_YIELD expr T_DOUBLE_ARROW expr { $$ = zend_ast_create(ZEND_AST_YIELD, $4, $2); }
	|	T_YIELD_FROM expr { $$ = zend_ast_create(ZEND_AST_YIELD_FROM, $2); }
```

从定义可以看出yield允许以下三种语法：
1. yield 
2. yield value
3. yield key => value

第一种没有写返回值，则默认返回值为NULL；第二种仅仅返回value，key则为自增的key；第三种返回自定义的key和value。

词法语法分析器扫描到yield会调用`zend_ast_create()`函数（Zend/zend_ast.c:135-144），得到类型（zend_ast->kind）为`ZEND_AST_YIELD`或者`ZEND_AST_YIELD_FROM`的zend_ast结构体。从**代码片段4.3.1**可以看出：`T_YIELD/T_YIELD_FROM`会被当成`expr_without_variable`，也就是表达式。接着，我们看看表达式的编译，在Zend/zend_compile.c:1794的`zend_compile_expr()`函数：

代码片段4.3.2：

```
void zend_compile_expr(znode *result, zend_ast *ast) /* {{{ */
{
	/* CG(zend_lineno) = ast->lineno; */
	CG(zend_lineno) = zend_ast_get_lineno(ast);

	switch (ast->kind) {
		case ZEND_AST_ZVAL:
			ZVAL_COPY(&result->u.constant, zend_ast_get_zval(ast));
			result->op_type = IS_CONST;
	// ...
		case ZEND_AST_YIELD: // 7272行
			zend_compile_yield(result, ast);
			return;
		case ZEND_AST_YIELD_FROM:
			zend_compile_yield_from(result, ast);
			return;
    // ...
}
/* }}} */
```
yield调用的`zend_compile_yield(result, ast)`函数，yield from调用的`zend_compile_yield_from(result, ast)`函数，这两个函数都会调用`zend_mark_function_as_generator()`，在Zend/zend_compile.c:1145：

代码片段4.3.3：

```C
static void zend_mark_function_as_generator() /* {{{ */
{
    /* 判断是不是函数/方法，不是就报错，也就是yield必须在函数/方法内 */
	if (!CG(active_op_array)->function_name) {
		zend_error_noreturn(E_COMPILE_ERROR,
			"The \"yield\" expression can only be used inside a function");
	}
	
	/* 如果有标识返回类型，则判断返回类型是否正确，只能是Generator及其父类(Traversable/Iterator) */
	if (CG(active_op_array)->fn_flags & ZEND_ACC_HAS_RETURN_TYPE) {
		const char *msg = "Generators may only declare a return type of Generator, Iterator or Traversable, %s is not permitted";
		if (!CG(active_op_array)->arg_info[-1].class_name) {
			zend_error_noreturn(E_COMPILE_ERROR, msg,
				zend_get_type_by_const(CG(active_op_array)->arg_info[-1].type_hint));
		}
		if (!(ZSTR_LEN(CG(active_op_array)->arg_info[-1].class_name) == sizeof("Traversable")-1
				&& zend_binary_strcasecmp(ZSTR_VAL(CG(active_op_array)->arg_info[-1].class_name), sizeof("Traversable")-1, "Traversable", sizeof("Traversable")-1) == 0) &&
			!(ZSTR_LEN(CG(active_op_array)->arg_info[-1].class_name) == sizeof("Iterator")-1
				&& zend_binary_strcasecmp(ZSTR_VAL(CG(active_op_array)->arg_info[-1].class_name), sizeof("Iterator")-1, "Iterator", sizeof("Iterator")-1) == 0) &&
			!(ZSTR_LEN(CG(active_op_array)->arg_info[-1].class_name) == sizeof("Generator")-1
				&& zend_binary_strcasecmp(ZSTR_VAL(CG(active_op_array)->arg_info[-1].class_name), sizeof("Generator")-1, "Generator", sizeof("Generator")-1) == 0)) {
			zend_error_noreturn(E_COMPILE_ERROR, msg, ZSTR_VAL(CG(active_op_array)->arg_info[-1].class_name));
		}
	}

	CG(active_op_array)->fn_flags |= ZEND_ACC_GENERATOR; // 标识函数是生成器类型！！！
}
/* }}} */
```
### 4.3.2 执行阶段
前两个特征都是在编译阶段，生成器函数编译完，得到的opcode为`DO_FCALL/DO_FCALL_BY_NAME`，解析opcode，得到对应的处理函数（handler）为`ZEND_DO_FCALL_BY_NAME_SPEC_HANDLER/ZEND_DO_FCALL_BY_NAME_SPEC_HANDLER`，这两个函数对于生成器处理基本是相同的，最终会调用`zend_generator_create_zval()`函数：

代码片段4.3.4：

```C
ZEND_API void zend_generator_create_zval(zend_execute_data *call, zend_op_array *op_array, zval *return_value) /* {{{ */
{
	zend_generator *generator;
	zend_execute_data *current_execute_data;
	zend_execute_data *execute_data;
	zend_vm_stack current_stack = EG(vm_stack); // 保存当前的vm_stack，以便后面恢复

	current_stack->top = EG(vm_stack_top);

	/* 先保存当前执行的execute_data，后面恢复 */
	current_execute_data = EG(current_execute_data);
	execute_data = zend_create_generator_execute_data(call, op_array, return_value); // 创建新的execute_data
	EG(current_execute_data) = current_execute_data; // 恢复之前的execute_data

	object_init_ex(return_value, zend_ce_generator); // 实例化Generator对象，赋给return_value，所以生成器函数返回的是Generator对象。 

    /* 如果当前执行的是对象方法，则增加对象的引用计数 */
	if (Z_OBJ(call->This)) {
		Z_ADDREF(call->This);
	}

	/* 把上面创建新的execute_data，保存到zend_generator */
	generator = (zend_generator *) Z_OBJ_P(return_value);
	generator->execute_data = execute_data;
	generator->stack = EG(vm_stack);
	generator->stack->top = EG(vm_stack_top);
	EG(vm_stack_top) = current_stack->top;
	EG(vm_stack_end) = current_stack->end;
	EG(vm_stack) = current_stack;

	/* 赋值给生成器函数返回值，真正是zend_generator，为了存储，转为zval类型，后面访问Generator类的时候会介绍 */
	execute_data->return_value = (zval*)generator;

	memset(&generator->execute_fake, 0, sizeof(zend_execute_data));
	Z_OBJ(generator->execute_fake.This) = (zend_object *) generator;
}
```

通过上面的代码片段可以知道：生成器调用时，函数的返回值返回了一个生成器对象，这就是上面提到的第三个特征。另外会申请自己的VM栈（vm_stack）跟原来的VM栈分离开来，互不干扰，每次执行生成器函数代码时只要修改executor_globals（EG）相应指针就可以切换到生成器函数自己的VM栈，这样就恢复到了生成器函数之前的状态。通常，execute_data在VM栈上分配（因为它实际上不进行任何内存分配，所以很快）。对于生成器，这不是最理想的，因为每次执行被暂停或恢复时都必须来回复制（相当大）的结构。 这就是为什么对于生成器，使用单独的VM栈分配执行上下文，从而允许仅通过替换指针来保存和恢复它。


## 4.4 yield生成值
[Generator对象的创建](#Generator对象的创建)中提到yield是一个表达式，
编译的时候最终会调用`zend_compile_yield()`函数，在Zend/compile.c:6337-6368：

代码片段 4.4.1：
```C
void zend_compile_yield(znode *result, zend_ast *ast) /* {{{ */
{
    // ...
    /* 编译key部分 */
	if (key_ast) {
		zend_compile_expr(&key_node, key_ast);
		key_node_ptr = &key_node;
	}
    /* 编译value部分 */
	if (value_ast) {
		if (returns_by_ref && zend_is_variable(value_ast) && !zend_is_call(value_ast)) {
			zend_compile_var(&value_node, value_ast, BP_VAR_REF);
		} else {
			zend_compile_expr(&value_node, value_ast);
		}
		value_node_ptr = &value_node;
	}
    /* 生成opcode为ZEND_YIELD的zend_op结构体，操作数1（OP1）为value ，操作数2（OP2）为key*/
	opline = zend_emit_op(result, ZEND_YIELD, value_node_ptr, key_node_ptr);

	// ...
}
```
从上面代码片段可以看出，yield对应的opcode是`ZEND_YIELD`，所以对应的处理函数为`ZEND_YIELD_SPEC_{OP1}_{OP2}_HANDLER`，生成的处理函数很多，但是代码基本都是一样的，都是由Zend/zend_vm_def.h中的`ZEND_VM_HANDLER(160, ZEND_YIELD, CONST|TMP|VAR|CV|UNUSED, CONST|TMP|VAR|CV|UNUSED)`生成的：
* 第一个参数160：ZEND_YIELD宏的值。
* 第二个参数ZEND_YIELD：opcode类型
* 第三个参数CONST|TMP|VAR|CV|UNUSED：表示操作数1（OP1，也就是值value）可以为这些类型的值。
* 第四个参数CONST|TMP|VAR|CV|UNUSED：表示操作数2（OP2，也就是键key）可以为这些类型的值。

Zend/zend_vm_execute.h（所有处理函数的存放文件）都是通过执行zend_vm_gen.php根据Zend/zend_vm_def.h的定义生成的。下面我们看一下这个定义函数：

代码片段 4.4.2：
```C
ZEND_VM_HANDLER(160, ZEND_YIELD, CONST|TMP|VAR|CV|UNUSED, CONST|TMP|VAR|CV|UNUSED)
{
    // ...
	/* 先销毁原来元素的key和value */
	zval_ptr_dtor(&generator->value);
	zval_ptr_dtor(&generator->key);

    /* 这部分是对value部分的处理 */
	if (OP1_TYPE != IS_UNUSED) { // 如果操作数1类型不是IS_UNUSED，也就是有返回值(yield value这类型)
		if (UNEXPECTED(EX(func)->op_array.fn_flags & ZEND_ACC_RETURN_REFERENCE)) {
		    // 前面一些判断，基本意思就是把值赋给generator->value，也就是生成值，这里就不贴代码了
		} else { // 如果不是引用类型
			// 根据不同的类型，把值赋给generator->value，也就是生成值，这里也不贴代码了
		}
	} else { // 如果操作数1类型是IS_UNUSED，也就是没有返回值(yield这类型)，则生成值为NULL
		ZVAL_NULL(&generator->value);
	}

	/* 这部分是对key部分的处理  */
	if (OP2_TYPE != IS_UNUSED) { // 如果操作数2类型不是IS_UNUSED，也就是有返回自定义的key(yield key => value这类型)
		// 根据不同的类型，把值赋给generator->key，也就是生成自定义的键，这里也不贴代码了

        /* 如果键的值类型为整型（IS_LONG）且大于当前自增key（largest_used_integer_key），则修改自增key为键的值*/
		if (Z_TYPE(generator->key) == IS_LONG
		    && Z_LVAL(generator->key) > generator->largest_used_integer_key
		) {
			generator->largest_used_integer_key = Z_LVAL(generator->key);
		}
	} else {
		/* 如果没有自定义key，则把下一个自增的值赋给key */
		generator->largest_used_integer_key++;
		ZVAL_LONG(&generator->key, generator->largest_used_integer_key);
	}

	if (RETURN_VALUE_USED(opline)) {
		/* If the return value of yield is used set the send
		 * target and initialize it to NULL */
		generator->send_target = EX_VAR(opline->result.var);
		ZVAL_NULL(generator->send_target);
	} else {
		generator->send_target = NULL;
	}

	/* 递增到下个op，这样下次继续执行就可以从下个op开始执行了 */
	ZEND_VM_INC_OPCODE();

	/* The GOTO VM uses a local opline variable. We need to set the opline
	 * variable in execute_data so we don't resume at an old position. */
	SAVE_OPLINE();

	ZEND_VM_RETURN(); // 中断执行
}
```

从上面代码片段可以看出：yield首先生成键和值（本质就是修改zend_generator的key和value），生成完键值后保存状态，然后中断生成器函数的执行。

## 4.5 Generator对象的访问
前面两小节介绍了Generator类和Generator对象的结构及创建，我们知道Generator对象可以通过foreach访问，也可以单独调用迭代器接口的方法（valid()/next()等）方法，参见**代码片段2.4.1**。

### 4.5.1 使用迭代器接口访问
先看看迭代器接口有哪些方法：
```php
Iterator extends Traversable {
    abstract public mixed current ( void )  // 返回当前元素 
    abstract public scalar key ( void )     // 返回当前元素的键
    abstract public void next ( void )      // 向前移动到下一个元素
    abstract public void rewind ( void )    // 返回到迭代器的第一个元素
    abstract public boolean valid ( void )  // 检查当前位置是否有效
}
```
对应C代码的函数如下：
```
rewind  -> ZEND_METHOD(Generator, rewind)
key     -> ZEND_METHOD(Generator, key)
next    -> ZEND_METHOD(Generator, next)
current -> ZEND_METHOD(Generator, current)
valid   -> ZEND_METHOD(Generator, valid)
```

ZEND_METHOD是内核定义的一个宏，方便阅读和开发，这里不做介绍，底层代码都在Zend/zend_generators.c:767-864。

#### 4.5.1.1 ZEND_METHOD(Generator, rewind)
ZEND_METHOD(Generator, rewind)
代码片段4.5.1：

```
ZEND_METHOD(Generator, rewind)
{
    // ...
	generator = (zend_generator *) Z_OBJ_P(getThis());
	zend_generator_rewind(generator);
}
```
`Z_OBJ_P(getThis())`，展开来是`(*(&execute_data.This)).value.obj`， 获取的是当前execute_data.This这个zval（类型为object）的object值（zval.value）的地址。但是这里强行转换是不是觉得很奇怪？

还记得**代码片段4.3.6**中提到：
> object_init_ex(return_value, zend_ce_generator); // 实例化Generator对象，赋给return_value，所以生成器函数返回的是Generator对象。 

初始化函数`object_init_ex()`最终会调用`_object_and_properties_init()`函数，在Zend/zend_API.c:1275-1310：

代码片段4.5.2：

```C
ZEND_API int _object_and_properties_init(zval *arg, zend_class_entry *class_type, HashTable *properties ZEND_FILE_LINE_DC) /* {{{ */
{
	// ...
	if (class_type->create_object == NULL) {
		ZVAL_OBJ(arg, zend_objects_new(class_type));
		if (properties) {
			object_properties_init_ex(Z_OBJ_P(arg), properties);
		} else {
			object_properties_init(Z_OBJ_P(arg), class_type);
		}
	} else {
		ZVAL_OBJ(arg, class_type->create_object(class_type));
	}
	return SUCCESS;
}
/* }}} */
```
从**代码片段4.4.2**可以看出，如果`zend_class_entry`定义有`create_object()`函数，那么会调用`create_object()`函数。而zend_ce_generator是有定义有`create_object()`函数，该函数为`zend_generator_create()`，参见[4.1 Generator类的注册及其存储结构](#Generator类的注册及其存储结构)：

代码片段4.5.3：

```C
static zend_object *zend_generator_create(zend_class_entry *class_type) /* {{{ */
{
    // ... 
	generator = emalloc(sizeof(zend_generator));
	memset(generator, 0, sizeof(zend_generator));
    // ...
	return (zend_object*)generator;
}
/* }}} */
```

内存里存储的是zend_generator，后面强制转换为zend_object，因为返回值要是zval类型，所以这里做了强制转换。这就能解释为什么可以`generator = (zend_generator *) Z_OBJ_P(getThis())`。

回到正题，`ZEND_METHOD(Generator, rewind)`得到zend_generator后，调用`zend_generator_rewind()`：

代码片段4.5.4：

```
static void inline zend_generator_rewind(zend_generator *generator)
{
	zend_generator_ensure_initialized(generator); // 保证generator已经初始化过了
    /* 如果已经yield过了，就不能再rewind */
	if (!(generator->flags & ZEND_GENERATOR_AT_FIRST_YIELD)) {
		zend_throw_exception(NULL, "Cannot rewind a generator that was already run", 0);
	}
}
```

如果yield过了，则不能再rewind，也就是不能再用foreach遍历，因为foreach也会调用rewind，这个后面再介绍。

#### 4.5.1.2 ZEND_METHOD(Generator, valid)
`ZEND_METHOD(Generator, valid)`，检查当前位置是否有效，如果无效，foreach会停止遍历。

代码片段4.5.5：
```C
ZEND_METHOD(Generator, valid)
{
    // ...
	generator = (zend_generator *) Z_OBJ_P(getThis());

	zend_generator_ensure_initialized(generator);

	zend_generator_get_current(generator);

	RETURN_BOOL(EXPECTED(generator->execute_data != NULL));
}
```
valid也是获取到zend_generator后，调用`zend_generator_get_current()`函数，获取当前需要运行的`zend_generator`，然后判断为`NULL`，以此已经更多的值生成了，这在[zend_generator结构体](#zend_generator结构体)中详细说明过。

#### 4.5.1.3 ZEND_METHOD(Generator, current)
`ZEND_METHOD(Generator, current)`获取当前元素的值。

代码片段4.5.6：
```C
ZEND_METHOD(Generator, current)
{
	// ...
	generator = (zend_generator *) Z_OBJ_P(getThis());

	zend_generator_ensure_initialized(generator);

	root = zend_generator_get_current(generator);
	if (EXPECTED(generator->execute_data != NULL && Z_TYPE(root->value) != IS_UNDEF)) {
		zval *value = &root->value;

		ZVAL_DEREF(value);
		ZVAL_COPY(return_value, value);
	}
}
```

和valid方法一样，也是先获取到zend_generator，然后判断生成器函数是否结束（`generator->execute_data != NULL`）并且有值（`Z_TYPE(root->value) != IS_UNDEF`），然后把值返回。

#### 4.5.1.4 ZEND_METHOD(Generator, key)
`ZEND_METHOD(Generator, key)`获取当前元素的键，也就是yield生成值时的key，没有指定会使用自增的key，即`zend_generator.largest_used_integer_key`。

代码片段4.5.7：

```C
ZEND_METHOD(Generator, key)
{
	// ...
	generator = (zend_generator *) Z_OBJ_P(getThis());

	zend_generator_ensure_initialized(generator);

	root = zend_generator_get_current(generator);
	if (EXPECTED(generator->execute_data != NULL && Z_TYPE(root->key) != IS_UNDEF)) {
		zval *key = &root->key;

		ZVAL_DEREF(key);
		ZVAL_COPY(return_value, key);
	}
}
```
跟`ZEND_METHOD(Generator, value)`差不多，`zend_generator.key`存储的就是当前元素的键，这在[zend_generator结构体](#zend_generator结构体)中详细说明过。

#### 4.5.1.5 ZEND_METHOD(Generator, next)
`ZEND_METHOD(Generator, next)`向前移动到下一个元素，也就是执行到下一个yield *。

代码片段4.5.8：

```C
ZEND_METHOD(Generator, next)
{
    // ...
	generator = (zend_generator *) Z_OBJ_P(getThis());

	zend_generator_ensure_initialized(generator);

	zend_generator_resume(generator);
}
```
主要分析`zend_generator_resume()`函数，这个函数比较重要：

代码片段4.5.9：

```C
ZEND_API void zend_generator_resume(zend_generator *orig_generator) 
{
	zend_generator *generator = zend_generator_get_current(orig_generator); // 获取要执行生成器

	/* 如果生成器函数已经结束，则直接返回，不能继续执行 */
	if (UNEXPECTED(!generator->execute_data)) {
		return;
	}

try_again: // 这个标签是个yield from用的，解析完yield from表达式，需要生成（yield）一个值。
    /* 如果有ZEND_GENERATOR_CURRENTLY_RUNNING标识，则表示已经运行，已经运行的不能再调用这方法继续运行 */
	if (generator->flags & ZEND_GENERATOR_CURRENTLY_RUNNING) {
		zend_throw_error(NULL, "Cannot resume an already running generator");
		return;
	}

	if (UNEXPECTED((orig_generator->flags & ZEND_GENERATOR_DO_INIT) != 0 && !Z_ISUNDEF(generator->value))) {
		/* We must not advance Generator if we yield from a Generator being currently run */
		return;
	}
    /* 如果values有值，说明是非生成器类的委托对象产生（yield from）的 */
	if (UNEXPECTED(!Z_ISUNDEF(generator->values))) {
		if (EXPECTED(zend_generator_get_next_delegated_value(generator) == SUCCESS)) { // 委托对象有值则直接返回
			return;
		}
		/* yield from没有更多值生成，则继续运行生成器函数后面的代码 */
	}

	/* Drop the AT_FIRST_YIELD flag */
	orig_generator->flags &= ~ZEND_GENERATOR_AT_FIRST_YIELD;

	{
		/* 保存当前执行的execute_data上下文和VM栈，以便后面恢复，这在前面已经介绍过了 */
		zend_execute_data *original_execute_data = EG(current_execute_data);
		zend_class_entry *original_scope = EG(scope);
		zend_vm_stack original_stack = EG(vm_stack);
		original_stack->top = EG(vm_stack_top);

		/* 修改执行器的指针，指向要运行的生成器函数和其相应的VM栈 */
		EG(current_execute_data) = generator->execute_data;
		EG(scope) = generator->execute_data->func->common.scope;
		EG(vm_stack_top) = generator->stack->top;
		EG(vm_stack_end) = generator->stack->end;
		EG(vm_stack) = generator->stack;

		// ...

		/* 执行生成器函数的代码 */
		generator->flags |= ZEND_GENERATOR_CURRENTLY_RUNNING;
		zend_execute_ex(generator->execute_data); // 执行，遇到yield停止继续执行
		generator->flags &= ~ZEND_GENERATOR_CURRENTLY_RUNNING;

		/* 修改VM栈相关的指针，因为上面运行过程中，VM栈不够，会重新申请新的MV栈，所以需要修改相关指针 */
		if (EXPECTED(generator->execute_data)) {
			generator->stack = EG(vm_stack);
			generator->stack->top = EG(vm_stack_top);
		}

		/* 恢复原来保存的execute_data上下文和VM栈 */
		EG(current_execute_data) = original_execute_data;
		EG(scope) = original_scope;
		EG(vm_stack_top) = original_stack->top;
		EG(vm_stack_end) = original_stack->end;
		EG(vm_stack) = original_stack;

		/* 处理异常，后面介绍throw()方法时再讲 */
		if (UNEXPECTED(EG(exception) != NULL)) {
			if (generator == orig_generator) {
				zend_generator_close(generator, 0);
				zend_throw_exception_internal(NULL);
			} else {
				generator = zend_generator_get_current(orig_generator);
				zend_generator_throw_exception(generator, NULL);
				goto try_again;
			}
		}

		/* yiled from没有生成值时，要重新进入(try_again)生成值 */
		if (UNEXPECTED((generator != orig_generator && !Z_ISUNDEF(generator->retval)) || (generator->execute_data && (generator->execute_data->opline - 1)->opcode == ZEND_YIELD_FROM))) {
			generator = zend_generator_get_current(orig_generator);
			goto try_again;
		}
	}
}
```
`zend_generator_resume()`函数，表面意思就是继续运行生成器函数。前面是一些判断，然后保存当前上下文，执行生成器代码，遇到yield返回，然后恢复上下文。至于yield后生成值并保存状态，

### 4.5.2 使用foreach访问
foreach访问生成器对象，其实就是调用`zend_ce_generator->get_iterator`，这在[Generator类的注册](#Generator类的注册)中介绍过，这是一个钩子，生成器用的是`zend_generator_get_iterator`，在Zend/zend_generators.c:1069-1093：

代码片段4.5.10：

```C
zend_object_iterator *zend_generator_get_iterator(zend_class_entry *ce, zval *object, int by_ref) /* {{{ */
{
	zend_object_iterator *iterator;
	zend_generator *generator = (zend_generator*)Z_OBJ_P(object);
	
	if (!generator->execute_data) {
		zend_throw_exception(NULL, "Cannot traverse an already closed generator", 0);
		return NULL;
	}

	if (UNEXPECTED(by_ref) && !(generator->execute_data->func->op_array.fn_flags & ZEND_ACC_RETURN_REFERENCE)) {
		zend_throw_exception(NULL, "You can only iterate a generator by-reference if it declared that it yields by-reference", 0);
		return NULL;
	}

	iterator = generator->iterator = emalloc(sizeof(zend_object_iterator)); // 申请内存

	zend_iterator_init(iterator); // 初始化

	iterator->funcs = &zend_generator_iterator_functions; //设置迭代器对象的相关处理函数
	ZVAL_COPY(&iterator->data, object);

	return iterator;
}
/* }}} */
```
前面是一些判断，重点是后面设置迭代器对象的相关处理函数为`zend_generator_iterator_functions`，来看看这个`zend_generator_iterator_functions`是什么：

代码片段4.5.11：

```C
static zend_object_iterator_funcs zend_generator_iterator_functions = {
	zend_generator_iterator_dtor,       // 销毁
	zend_generator_iterator_valid,      // 判断当前位置是否有效
	zend_generator_iterator_get_data,   // 获取当前元素
	zend_generator_iterator_get_key,    // 获取当前元素的键
	zend_generator_iterator_move_forward, // 向前移动到下一个元素
	zend_generator_iterator_rewind      // 指向第一个元素
};
```

（未完待续）