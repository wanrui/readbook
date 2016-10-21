[TOC]  
# lua 学习上篇
## 开始
lua 的独立解释程序 + lua脚本即可运行。学习过程中推荐使用独立的lua解释程序运行  

1. 命令  
   * 退出交互模式和解释器方法为end-of-file控制符，或者系统exit函数os.exit()。  
   * lua -i 运行完指定的程序块后进入交互模式。  
   * dofile("hello.lua")加载后可以进行测试。  
2. 语法
   * 保留字 及标识符 大小写敏感。  
     保留字（' end function in local nil not repeat then '）
   * 注释函数方法  
    单行注释--  
    块注释--[[  --]] 取消注释 可以在前面加‘-’即可取消注释  
   * 未初始化的变量的访问不会引发错误，输出为nil ，同时删除一个全局变量直接设置为nil，如果生命周期比较短的那种变量建议使用local 局部变量。
3. 类型与值  
lua 是一种动态类型的语言 语言中没有定义类型的语法每个值都包含类型信息  
   * 8种基础类型  
        nil boolean number string userdata function thread table  
        ```lua 
        print(type('hello'))	-->string
        ```
   * string 字符串，是不可变的值，不能像C语言那样直接修改某个字符，而是根据修改要求新建一个字符串。
        ```lua
        a ='one string'
        b = string.gsub(a,'one',another)
        print(a)
        print(b)
        ```
   * 字符串赋值过程中如果需要屏蔽转义，可以使用 [[ 符号
        * `#`符号可以表示字符串的长度
   * table （表）  
        具有特殊索引方式的数组。不仅可以通过整数索引，还可以通过字符串或者其他类型的值进行索引，*无固定大小*，可以动态添加任意元素到一个talbe中。
        lua不会暗中生成table 的副本。table的创建都是通过构造表达是完成的。最简单的表达式就是{}  
        * table 数组通常以1位开始索引 ，可用#返回一个数组的最后一个索引值。  
        * table.maxn 可以返回table 最大索引  

            ```lua
            a = {}
            k = 'x'
            a[k] = 10
            a[20] = 'great'
            print(a['x'])
            a['x'] = a['x'] + 1
            print(a['x'])
            print(a[20])
            print(table.maxn(a))
            ```  
   * function （表）  
    lua 既可以调用自己写的函数，也可以调用C语言写的函数 
## 流程控制  

### 逻辑表达式  
1. 算数  
取模操作是lua5.1 后支持的操作
  ``` lua
    a = 10 
    b = 3 
    print(a%b)
    print(math.floor(a/b))
    c = a - math.floor(a/b)*b
    print(c)
    print( c == a%b )
  ```
2. 关系  
* 对应的关系操作符如下 == ~= >= <= > < 。特别注意的地方是不等于使用的符号为~=  
* 数据做比较的时候首先比较的数据类型。  
* table 和userdata 类型的数据进行比较的时候，比较的是引用比较

    ```lua
        a = {}
        b = {}
        a.x = 1
        a.y = 1
        b.x = 1
        b.y = 1

        print (a == b)
        c = b 
        print (b ==c )
    ```

3. 逻辑  
    逻辑操作表达式 and or not
4. 字符串连接  
    字符串的链接操作 使用.. 如果其中一个是字符串的话那么 会将数字转化成字符串。由于字符串是不可更改的，在使用连接符号后会生成一个新的字符串。

5. 优先级
6. table构造  
* 构造方式有很多种，是key 和value 的模式进行构造。
* 同时还提供了一种方式。
``` lua
 days = {'sunday','monday','tuesday','wednesday',
 'thursday','friday','saturday'}
 print(days[4]) -- -->wednesday

 a = {x = 1 , y =2}
 b = {}
 b.x = 1;
 b.y = 2; 
 b.x =nil --删除x字段

---链表
list = nil 
for line in io.lines() do
    if(line ~= "exit") then
         list = {next = list ,value =line}
    else
        break 
    end
end

local l = list 
while l  do 
    print(l.value)
    l  = l.next 
end

--以上的构造方法中有两种限制，不能使用负数和运算发作为key。为了满足这些要求
--lua 提供一种更加通用的格式。
opnames = {["+"] = "add",["-"] = sub}
c = {[1] = 'wanrui',[2] = 'hello'}  
```
### 流程控制
* 多重赋值  
``` lua  
a, b = 2 ,3 
print('a-->'..a)
print('b-->'..b)
a,b = b,a
print('a-->'..a)
print('b-->'..b)
```  
* 全局变量与局部变量 
``` lua
x =10
i = 1
while( i<x ) do
    local x = 2*i
    print('while local x-->'..x)
    i = i +1;
end 
print('global x-->'..x)
--[[ 输出结果
while local x-->2
while local x-->4
while local x-->6
while local x-->8
while local x-->10
while local x-->12
while local x-->14
while local x-->16
while local x-->18
global x-->10
--]]
```
* 所有的控制结构都是用的是end 作为结果，但是只有repeat 使用util作为结尾
* repeat 和while 的区别是，while 先判断条件是不是符合，repeat 先执行然后在判断  
* repeat 与其他不同的是局部变量的作用域包含了条件测试。
1. if then else
``` lua
function toString(ops)
    if(ops == '+') then
        print('plus')
    elseif(ops == '-') then 
        print('minus')
    elseif(ops == '*') then 
        print('multi')
    elseif(ops == '/') then 
        print('div')
    end
    return ops
end

toString('+')
toString('-')
toString('*')
toString('/')
--[[
plus
minus
multi
div
    --]]
```  
2. while  
``` lua
x =10
i = 1
while( i<x ) do
    local x = 2*i
    print('while local x-->'..x)
    i = i +1;
end 
print('global x-->'..x)
--[[ 输出结果
while local x-->2
while local x-->4
while local x-->6
while local x-->8
while local x-->10
while local x-->12
while local x-->14
while local x-->16
while local x-->18
global x-->10
--]]
```
3. repeat  
``` lua
function fact(a)
    local i = a
    local result = 1
    repeat
        result = result * i
        i = i - 1 
        print('repeat -->' .. result)
    until  i <= 0

    print(result)
end 
fact(3)
--[[

repeat -->3
repeat -->6
repeat -->6
6
    --]]
```
4. for（数字型）  
for语句的循环控制是 exp1变化到exp2，中间的步长为exp3。exp3是可选想如果不选的话默认步长为1。可以使用math.huge来标识不设置上限制。
``` lua
--[[
for var=exp1,exp2,exp3 do
    <执行>
end
for-->1
for-->3
for-->5
for-->7
for-->9
--]]
for var=1,10,2 do
    print('for-->' .. var)
end
```

5. 泛型for 与迭代器  
通过迭代器函数进行遍历  

``` lua
days = {'sunday','monday','tuesday','wednesday',
 'thursday','friday','saturday'}

 for i,v in ipairs(days) do
    print('for iterator -->'..i..v)
 end
 --[[
 for iterator -->1sunday
for iterator -->2monday
for iterator -->3tuesday
for iterator -->4wednesday
for iterator -->5thursday
for iterator -->6friday
for iterator -->7saturday
     --]]
```
6. break & return

## 函数
### 函数定义
1. 多重返回值
2. 变长参数
3. 具名实参
### 输入函数
1. closure
2. 非全局函数
3. 正确的尾调用

## 编译、执行、错误
1. 编译 
2. c代码
3. 错误
4. 错误处理与异常
5. 错误消息与追溯

## 协同程序
1. 协同程序基础
2. 管道与过滤器
3. 以协同程序实现迭代器
4. 非抢先式的多线程

## 完成示例

## 数据结构
1. 数组 
2. 矩阵与多维度数组 
3. 链表
4. 队列与双向队列 
5. 集合与无序组 
6. 字符串缓冲 
7. 图  

## 数据文件与持久性
1. 数据文件
2. 串行化

## 元表（metable）与元方法（meatmethod）  
1. 算数类的原方法
2. 关系类的元方法
3. 库定义的元方法
4. table 访问的元方法

## 环境 
1. 具有动态名字的全局变量
2. 全局变量声明
3. 非全局变量

##模块与包
1. require函数 
2. 编写模块的基本方法
3. 使用环境
4. module函数
5. 子模块与包

## 面向对象编程  
1.  类
2. 继承
3. 多重继承
4. 私密性
5. 单一方法（single-method）做法

## 弱引用talbe
1. 备忘录函数（memoize） 
2. 对象属性  
3. 回顾table 的默认值


# lua 学习下篇

## 数学库  

## table 库 
1. 插入和删除
2. 排序
3. 连接  

## 字符串库  
1. 基础字符串函数  
2. 模式匹配（pattern-matching）函数
  * string.find  
  * string.match 
  * string.gsub 
  * string.gmatch
3. 模式 
4. 捕获
5. 替换
6. 技巧

## I/O库  
1. 简单IO模型
2. 完整IO模型

## 操作系统库
1. 日期和时间
2. 其他系统调用

## 调试库
1. 自省机制
2. 钩子 
3. 性能剖析

## CAPI 概述 
1. 示例
2. 栈 
3. 错误处理 

## 扩展应用程序 
1. 基础 
2. table操作 
3. 调用lua函数 
4. 一个通用的调用函数 

## 从lua调用c
1. c 函数 
2. c 模块 

## 编写C 函数的技术
1. 数组的操作 
2. 字符串的操作 
3. 在C函数中保存状态
 * 注册表
 * c函数的环境 
 * upvalue  

## 用户自定义类型  
1. Userdata 
2. 元表 
3. 面向对象的访问 
4. 数组访问  
5. 轻量级userdata

## 资源管理
1. 目录迭代器
2. xml 分析器

## 线程和状态
1. 多个线程
2. lua状态 

## 内存管理 
1. 分配函数 
2. 垃圾收集器
  * 原子操作
  * 垃圾收集器的API





