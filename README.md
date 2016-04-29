# tabtoy

游戏客户端,服务器的策划表格数据导出

将电子表格文件根据制定proto及字段转化规则,导出为Protobuf文本格式(*.pbt)

使用者通过读取Protobuf文本格式直接获取到所有的格式化数据

# 优点

本转化器无需依赖vbs,vba.跨平台

直接输出基于Protobuf文本的格式化数据, 直接读取

字段位置随意调整, 自动检查错误, 精确报错位置

充分利用CPU多核进行导出, 是已知的现有导出器中最快的


# 迭代历程

* 2011年: 一代导出器,基于VBA的表格内建导出器,速度慢,复用困难,容易错,不安全

* 2012年: 二代导出器,基于C++和Protobuf的导出器,内容格式与导出器混合编写,需要vbs导出csv,速度慢

* 2013年: 三代导出器,在二代基础上做到内容格式与导出器独立,但依然依赖csv前置导出,增加逗号分隔格子内容,导出速度慢

* 2015年: 四代导出器,基于Golang导出器,增加ID重复检查,数组格的多重写法, 支持a.b.c栏位导出, 导出速度大大提高

* 2016年: 五代导出器,在四代基础上重构,开源,支持并发导出,速度达到极致

# 应用情况

前面多个版本都在本人项目中使用

53个Excel源文件, 格式xlsm, 大小3.8M

导出速度

* 9.4s 第四代导出器

* 4.9s 第五代导出器单线程

* 2.4s 第五代导出器i7-4790 8核并发


# 编译

	go get github.com/davyxu/tabtoy
	
	go install github.com/davyxu/tabtoy


# 导出步骤

## 第一步: 定义协议描述表结构

```protobuf
syntax = "proto3";

package test;

enum ActorType
{
	Fighter = 0;	// [tabtoy] Alias:"格斗士"  #使用#号可在meta后方添加注释
	Power = 21;		// [tabtoy] Alias: "超能"
}


message Prop
{
	int32 HP = 1;
	float AttackRate  = 2;
}

// 代表每一行对应的整个字段描述
message ActorDefine
{
	// 唯一ID
	int32 ID = 1; 	// [tabtoy] RepeatCheck: true #ID重复检查
	
	// 角色名称
	string Name = 5; 
	
	Prop Struct  = 10;
	
	repeated int32 BuffID = 20;	// 重复字段可以用多列进行读取
	
	// 角色类型
	ActorType Type = 30; 

	repeated int32 SkillID = 40; // [tabtoy] String2ListSpliter: "," #使用,切割字符串
	
	Prop StrStruct = 50; // [tabtoy] String2Struct: true
		
}

// 代表导出文件
message ActorFile
{
	repeated ActorDefine Actor = 1; // 每一个元素代表电子表格的一行
}

```

## 第二步: 编写电子表格字段及列头
![电子表格](doc/table.png)


## 第三步: 通过 批处理/shell 填写导出目标及信息

# 通过protobuf的编译器protoc, 配合protoc-gen-meta插件导出带有meta信息的pb文件

```BAT
..\proto\protoc.exe test.proto --plugin=protoc-gen-meta=..\..\..\..\..\bin\protoc-gen-meta.exe --proto_path "." --meta_out test.pb:.
```

# 通过tabtoy读取meta信息及电子表格,指定输出文件夹,输出同名的pbt文件
```BAT
..\..\..\..\..\bin\tabtoy.exe --mode=xls2pbt --pb=test.pb --outdir=. Actor.xlsx

```

导出文件样式
```txt
# Generated by github.com/davyxu/tabtoy
Actor {ID: 100 Name: "黑猫警长" Struct {HP: 100 AttackRate: 0.6} BuffID: 0 BuffID: 0  SkillID: 4 SkillID: 6 SkillID: 7}
Actor {ID: 101 Name: "葫芦娃" Struct {AttackRate: 0.8} BuffID: 3 BuffID: 1 Type: Power SkillID: 1}
Actor {ID: 102 Name: "舒克" Struct {AttackRate: 0.7} BuffID: 0 BuffID: 0  SkillID: 0 StrStruct {HP: 2 AttackRate: 0.5}}
Actor {ID: 103 Name: "贝塔" Struct {HP: 205} BuffID: 0 BuffID: 0  SkillID: 0}
Actor {ID: 104 Name: "邋遢大王" Struct {AttackRate: 1} BuffID: 0 BuffID: 0  SkillID: 0}


```

# Protobuf Text格式(pbt)
Google Protobuf 官方支持的格式
可通过官方protobuf库读取,写入这种格式
与json区别在于: 
json的字段名必须是带双引号, 且数组需要用[]圈住, 多重字段尾部必须加逗号
* 格式
	key1: value1  key2: value2
	冒号组合key,value,  空格分隔字段
	
	\#作为注释

# Proto文件规则
proto文件格式范例参考test/test.proto
需要配合github.com/davyxu/pbmeta的protobuf插件protoc-gen-meta导出proto文件的meta信息

在proto的字段后方的注释中以[tabtoy]开头的注释将被理解为meta信息, 用于描述字段导出功能修饰
例如: // [tabtoy] RepeatCheck: true #ID重复检查

[tabtoy] 后方的格式为Protobuf Text, 使用#作为注释

具体的meta功能请参考后面的小节


# 电子表格文件头格式

## 导出头

必须放在需要导出的Sheet的1,1位置

格式:
	ProtoTypeName: "package名.message名"  RowFieldName: "导出字段"

## Proto字段行

Proto字段列, 必须放在第二行

## 功能描述行

用于描述字段功能, 必须放在第三行

## 导出数据行

从第四行开始, 一直到 **第一列为空** 时表示数据行结束, 后续行不再导出

# 电子表格Proto字段行基本功能

## 多层结构字段

当字段类型为结构体A, 实例名为a时, 若需要设置结构体内部的某字段类型B, 实例名为b时

只需要在Proto字段行中填写a.b即可

本功能仅限于非repeated字段

## 多重字段的分单元格导出

当字段类型为repeated类型时, 可将字段按同名方式切成多个字段导出

## 字符串转义
\r 和 \n 对应的转义符将被转成字符 \r 和 \n  在pbt加载时, 加载器会自动转回转义符


# 电子表格Proto字段行meta扩展功能

## 枚举别名

当枚举值字段meta信息中填写Alias: "别名"时

单元格值中即可填写原枚举的程序枚举类型, 也可以填写别名


## 字符串方式导出多层结构字段

格式: String2Struct: true

当字段meta信息中填写String2Struct: true时, 将把单元格的值理解为Protobuf Text格式并检查输出

## 字段重复性检查

格式: RepeatCheck: true

当字段meta信息中填写RepeatCheck: true时, 将检查列中的本字段是否有重复字段(字符串方式)


## 字段值切割

格式: String2ListSpliter:  ";"

当字段类型为repeated类型时, 且字段meta信息中填写String2ListSpliter: "分隔字符串"时

字段值会用"分隔字符串"切割并存储到repeated字段中

P.S. 建议分割字符串不要使用逗号. 电子表格中, 数字中带逗号表示数位分隔符, 会导致无法切割的问题




## 默认值功能
格式: DefaultValue: "xxx"

protobuf v3版本中去掉了DefaultValue支持, 在meta信息中添加DefaultValue可以在单元格为空时
自动填充为DefaultValue中的值, DefaultValue为字符串, 内部导出处理方式与单元格流程一致





# 其他规则
* 建议每个proto文件对应1个电子表格(xls,xlsm),名字统一

	例: Actor.proto 对应的电子表格名Actor.xls,最终导出文件为Actor.pbt

* 一个电子表格内的所有Sheet都是合并导出为一个pbt
	例: Actor.xls中拥有名为ActorA ActorB的Sheet,最终导出的Actor.pbt中将包含ActorA和ActorB的数据

* 没有导出头(单元格1,1)的格子不会被导出



# 使用导出文件

## Golang
引用github.com/golang/protobuf库

例子:

```golang



	func LoadPBTFile(filename string, msg proto.Message) error {
	
		content, err := ioutil.ReadFile(filename)
	
		if err != nil {
			return err
		}
	
		return proto.UnmarshalText(string(content), msg)
	}


	// 准备你的消息结构
	var msg gamedef.YourMessage

	// 直接读取pbt文件
	LoadPBTFile( "Data.pbt", &msg )
	
	
```

## C# 

步骤

* 使用github.com/davyxu/mergepbt 将多个pbt合并并转为二进制

* 使用protobuf-net库加载对应的二进制文件并读取

例如:

```csharp

        using (var stream = File.OpenRead(pathToPbt))
        {
            var yourmsg = ProtoBuf.Serializer.Deserialize<gamedef.ClientConfig>(stream);
            stream.Close();
        }   


```

## 其他语言

* 支持protobuf文本格式的语言可以直接读取pbt文件

例如: 官方包内的支持的C++, C#, Java, Golang等等

* 不支持文本格式的语言需要使用github.com/davyxu/mergepbt进行合并转换, 使用二进制读取

例如: Unity3D使用的protobuf-net



# FAQ
* 为什么输出文本格式,而不是二进制?

	从设计角度: 文本只需要字段就可以导出, 而二进制需要复杂的二进制连接操作,设计复杂度较低
	
	从使用角度: 文本调试,查看很方便
	

* 为什么只有Protobuf Text输出没有json或者其他格式?

	Protobuf 2.X时是对Protobuf Text有良好的支持, 包括Golang
	
	进入3.0 后, 大概由于Protobuf Text不是主流格式, 因此官方还是提供json格式支持
	

* C++和C#支持么?

	C++用官方的Protobuf库可以读取导出格式, C#的protobuf-net无法读取, 需要自己根据你的消息格式转换文本到二进制才可读取
	

* 导出meta信息时, 多个proto文件写的批处理很长, 怎么办?

	批处理的多行连接符是^, 例如
	
	protoc a.proto ^
	
	b.proto ^
	
	c.proto
	
	注意 空格不能少
	

* 为什么并发导出时, 日志顺序是乱的?

	Visual Studio并发编译C++时也是乱的, 当然它可以调顺序模式
	
	乱和速度不可兼得, 因为懒:)
	
# 下载

不定期更新打包版本
https://github.com/davyxu/tabtoy/releases

想获取最新版请取最新代码编译

# TODO




# 备注

感觉不错请star, 谢谢!

博客: http://www.cppblog.com/sunicdavy

知乎: http://www.zhihu.com/people/xu-bo-62-87

邮箱: sunicdavy@qq.com
