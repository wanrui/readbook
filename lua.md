# lua 学习
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
