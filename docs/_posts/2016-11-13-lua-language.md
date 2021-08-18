---
title: Lua 语言
date: 2016-11-13 09:30:09
categories:
  - 程序语言
tags:
  - Lua
---

最近在忙着做地理方面的机器学习的研究，把博客给荒废了。今天就把最近常用的Lua编程语言做些笔记，方便以后参考。

[Lua](http://www.lua.org/) 是一门动态类型的脚本语言，具有很强的拓展性，可以和C/C++进行交互。Lua的语法简单、运行十分高效，通常应用于嵌入式程序、C程序，当然也可以单独使用Lua进行程序设计。

<!-- more -->

## 安装 Lua
* windows
使用 [LuaDist](http://luadist.org/) 进行安装

* linux
```
curl -R -O http://www.lua.org/ftp/lua-5.3.3.tar.gz
tar zxf lua-5.3.3.tar.gz
cd lua-5.3.3
make linux test
```

* mac
```
curl -R -O http://www.lua.org/ftp/lua-5.3.3.tar.gz
tar zxf lua-5.3.3.tar.gz
cd lua-5.3.3
make macosx test
```

安装完成后，你可以在命令行中输入 lua 命令进入解释器界面进行编码尝试；也可以在一个文本编辑器中书写代码，保存成文件，比如保存成test.lua，然后再命令行中输入 lua test.lua 执行文件中的代码。

下面就开始介绍 Lua 语言啦。

## 注释
* 单行注释
```lua
-- 两个短横线表示一个单行注释
```

* 块注释
```lua
--[[
    使用两个短横线和两个方括号
    表示多行注释
--]]
```

## 基本变量
* nil
表示值未定义
```lua
t = nil
```

* boolean
```lua
t = false
```

* number
Lua中所有的数都是double类型的
```lua
num = 42
```

* string
```lua
s1 = '通常用单引号表示字符串'
s2 = "双引号也可以表示字符串"
s3 = [[ 两个方括号
        表示多行的
        字符串 ]]
```
## 流程控制
* if 语句
```lua
if num > 40 then
    print('over 40')
elseif s ~= 'abc' then
    io.write('s is not equal to abc')
else
    print('num <= 40 and s == abc')
end
```

* while 语句
```lua
while num < 50 do
    num = num + 1
end
```

* for 语句
```lua
for i = 0, 10 do
    print(i)
end

for j = 100, 1, -1 do
    print(j)
end
```

* and or 语句，类似于其它语言的三元操作符：a?b:c
```lua
ans = aBoolValue and 'yes' or 'no'
```

* repeat until 语句
```lua
repeat
    print(num)
    num = num - 1
until num == 0
```

## 方法(Functions)
```lua
function fib(n)
    if n < 2 then return n end
    return fib(n - 2) + bib(n - 1)
end

--闭包和匿名方法
function adder(x)
    return function(y) return x + y end
end

a1 = adder(9)
a2 = adder(36)
print(a1(16))  --> 25
print(a2(64))  --> 100

-- 方法也是一种类型，可以被赋值
f = function (x) return x * x end
```

## 表（Tables）
Tables是Lua唯一的也是最重要的数据结构，Tables实际上是关联数列。Table中的元素都以键值对的形式组织。

```lua
-- 一般的键是字符串
t = {key1 = 'value1', key2 = false}

-- 字符串的键可以通过点操作取值
print(t.key1)  -- Prints 'value1'.

--任何非nil的值都可以作为键
u = {['@!#'] = 'qbert', [{}] = 1729, [6.28] = 'tau'}

--通过方括号的形式去取值
print(u[6.28])  -- prints "tau"

-- 若不指定键，通常会用默认的int表示，通过这种方式可以表示数组
v = {'value1', 'value2', 1.21, 'gigawatts'}
for i = 1, #v do  -- #v is the size of v for lists.
  print(v[i])  -- Indices start at 1 !! SO CRAZY!
end
```

## Metatables and metamethods
一个Table可以包含一个Metatable来重载操作符行为。
下面显示了如何表示复数的相加：
```lua
f1 = {a = 1, b = 2}  -- Represents the fraction a/b.
f2 = {a = 2, b = 3}

-- This would fail:
-- s = f1 + f2

metafraction = {}
function metafraction.__add(f1, f2)
  local sum = {}
  sum.b = f1.b * f2.b
  sum.a = f1.a * f2.b + f2.a * f1.b
  return sum
end

setmetatable(f1, metafraction)
setmetatable(f2, metafraction)

s = f1 + f2  -- call __add(f1, f2) on f1's metatable
```
可以通过metatable来定义默认的键值对
```lua
-- An __index on a metatable overloads dot lookups:
defaultFavs = {animal = 'gru', food = 'donuts'}
myFavs = {food = 'pizza'}
setmetatable(myFavs, {__index = defaultFavs})
eatenBy = myFavs.animal  -- works! thanks, metatable
```

## 类和继承
Lua没有类的对象，但是可以通过Table和Metatable来构造。

Dog类
```lua
Dog = {}

function Dog:new()
    local newObj = {sound = 'woof'}
    self.__index = self
    return setmetatable(newObj, self)
end

function Dog:makeSound()
    print('I say ' .. self.sound)
end

mrDog = Dog:new()
mrDog:makeSound() -- 'I say woof'
```
Dog:new 和Dog:makeSound这种冒号调用方法的方式可以隐式地传递当前调用对象进入函数，也就是函数体中引用的self。


继承Dog类
```lua
LoundDog = Dog:new()
function = LoundDog:makeSound()
    local s = self.sound .. ' '
    print(s .. s .. s)
end

seymour = LoundDog:new()
seymour:makeSound() -- 'woof woof woof'
```
LoudDog 得到了 Dog 的变量和函数，并且重新定义了makeSound这个函数。

## Modules
Modules用来分离和共享代码。
Modules一般返回一个Table对象。

```lua
-- mymod.lua
local M = {}

local function sayMyName()
  print('Hrunkner')
end

function M.sayHello()
  print('Why hello there')
  sayMyName()
end

return M
```

另一个文件可以调用mymod.lua这个模块

```lua
local mod = require('mymod')  -- Run the file mod.lua.

-- This works because mod here = M in mod.lua:
mod.sayHello()  -- Says hello to Hrunkner.

-- This is wrong; sayMyName only exists in mod.lua:
mod.sayMyName()  -- error
```

最后，如果对 Lua 感兴趣，想进一步学习的话，可以下载 [Programming in Lua](https://pan.baidu.com/s/1qYPH7M8) 进行学习。