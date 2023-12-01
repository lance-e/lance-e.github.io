# viper:Go语言配置管理神器


<!--more-->

# viper:Go语言配置管理神器

参考文档：[viper](https://github.com/spf13/viper/blob/master/README.md)

文档汉化：[李文周的博客](https://www.liwenzhou.com/posts/Go/viper/)

##### 一.安装

~~~go
go get github.com/spf13/viper
~~~

##### 二.支持特性

1.设置默认值

2.从json ,toml,yaml,hcl,envfile,java properties格式的配置文件读取配置信息

3.实时监控和重新读取配置文件(optional)

4.从环境变量中读取

5.从远程配置系统（etcd和Consul）读取并监控配置变化

6.从命令行参数读取配置

7.从buffer读取配置

8.显示配置值

##### 四.viper可以做什么？

1.查找，加载和反序列化json,toml,yaml,hcl,ini,envfile,java properties格式的配置文件。

2.提供一种机制为你不同配置选项指定选项的值。

3.提供一种机制来通过命令行参数覆盖指定选项的值。

4.提供别名系统，一边在不破坏现有代码的情况下轻松重命名参数。

5.当用户提供与默认值相同的命令行或配置文件时，可以很容易的分辨出它们之间的区别。

##### 五.viper优先级

(从高到低)

1.显式调用Set设置值

2.命令行参数（flag）

3.环境变量

4.配置文件

5.key/value存储

6.默认值

##### 六.将值存入viper

###### 1.设置默认值：

~~~go
viper.SetDefault("name", "lance")//设置name默认值为lance
value := viper.GetString("name") //获取默认值
fmt.Println(value)//打印lance
~~~

###### 2.读取配置文件：

viper可以搜索多个路径，但是目前**单个**viper实例只支持**单个**配置文件。

viper不默认任何配置搜索路径，将默认决策留给应用程序。

搜索配置文件：

*不需要任何特定的路径，但是**至少应该提供一个***配置文件预期出现的路径

配置文件有扩展名：

~~~go
viper.SetConfigFile("./config.yaml")//通过指定配置文件路径
~~~

配置文件无扩展名

~~~go
viper.SetConfigName("config1")//配置文件名称（本身就没有扩展名）
viper.SetConfigType("yaml")//没有扩展名，必须配置此项
viper.AddConfigPath(".")//搜索配置文件所在的路径
viper.AddConfigPath("./...")//多次调用以添加多个搜索路径
...
~~~

读取配置文件：

~~~go
	//说明： 这里执行viper.ReadInConfig()之后，viper才能确定到底用哪个文件
	err := viper.ReadInConfig() // 查找并读取配置文件
	if err != nil {
		if err == err.(viper.ConfigFileNotFoundError) {
			panic(err) // 处理找不到配置文件的错误
		}
		panic(err)
	}
~~~

*如果有多个同名配置文件，扩展名优先级：*

json>toml>yaml>yml>properties(java中的配置文件名)>props(java中的配置文件名)

###### 3.写入配置文件：

已经预定义路径：

~~~go
	//若没有预定义路径会报错
	viper.WriteConfig()//覆盖存在的配置文件
	viper.SafeWriteConfig()//不会覆盖存在的配置文件，配置文件不存在就创建配置文件
~~~

给定路径：

~~~go
	viper.WriteConfigAs()//覆盖当前配置文件
	viper.SafeWriteConfigAs()//不会覆盖存在的配置文件，配置文件不存在就创建配置文件
~~~

###### 4.监控并重新读取配置文件：

viper支持在运行时实时读取配置文件的功能

~~~go
//之前已经添加好了配置路径
viper.WatchConfig()
viper.OnConfigChange(func(e fsnotify.Event) {
  // 配置文件发生变更之后会调用的回调函数
	fmt.Println("Config file changed:", e.Name)
})

~~~

###### 5.从io.Reader读取配置：

实现自己所需的配置原并将其提供给viper

~~~go
	viper.SetConfigFile("./config.yaml")
	config := []byte(`
hello : world
ping: pong
`)
	err := viper.ReadConfig(bytes.NewBuffer(config))
	if err != nil {
		panic(err)
	}
	//只是读取，未写入配置文件
	value := viper.Get("ping")
	fmt.Println(value)
~~~

###### 6.覆盖设置：

这些可能来自命令行标志，也可能来自你自己的应用程序逻辑

~~~go
viper.Set("Verbose", true)
viper.Set("LogFile", LogFile)
~~~

###### 7.注册和使用别名：

别名允许多个键引用单个值

~~~go
viper.RegisterAlias("loud", "Verbose")  // 注册别名（此处loud和Verbose建立了别名）

viper.Set("verbose", true) // 结果与下一行相同
viper.Set("loud", true)   // 结果与前一行相同

viper.GetBool("loud") // true
viper.GetBool("verbose") // true
~~~

###### 8.使用环境变量

###### 9.使用Flags

###### 10.远程Key/Value存储支持

###### 11.监控etcd中的更改-未加密

(以后有待补充)

##### 七.从viper获取值

常用根据值类型获取值的方法：

~~~go
Get(key string) : interface{}
GetBool(key string) : bool
GetFloat64(key string) : float64
GetInt(key string) : int
GetIntSlice(key string) : []int
GetString(key string) : string
GetStringMap(key string) : map[string]interface{}
GetStringMapString(key string) : map[string]string
GetStringSlice(key string) : []string
GetTime(key string) : time.Time
GetDuration(key string) : time.Duration
IsSet(key string) : bool //判断键是否存在
AllSettings() : map[string]interface{}
~~~

###### 1.访问嵌套的键

访问器方法也接受深度嵌套键的格式化路径。例如，如果加载下面的JSON文件：

~~~json
{
    "host": {
        "address": "localhost",
        "port": 5799
    },
    "datastore": {
        "metric": {
            "host": "127.0.0.1",
            "port": 3099
        },
        "warehouse": {
            "host": "198.0.0.1",
            "port": 2112
        }
    }
}
~~~

viper通过传入 . 分隔的路径来访问嵌套字段：

~~~go
//已经读取了配置文件
viper.GetString("datastore.metric.host")//返回127.0.0.1
~~~

如何存在键中包含 " . "的情况，并且与分隔的键路径匹配，那么返回这个键的值：

~~~json
{
    "datastore.metric.host": "0.0.0.0",
    "host": {
        "address": "localhost",
        "port": 5799
    },
    "datastore": {
        "metric": {
            "host": "127.0.0.1",
            "port": 3099
        },
        "warehouse": {
            "host": "198.0.0.1",
            "port": 2112
        }
    }
}

viper.GetString("datastore.metric.host") // 返回 "0.0.0.0"
~~~

（文档原文）：

这遵守上面建立的优先规则；搜索路径将遍历其余配置注册表，直到找到为止。(译注：因为Viper支持从多种配置来源，例如磁盘上的配置文件>命令行标志位>环境变量>远程Key/Value存储>默认值，我们在查找一个配置的时候如果在当前配置源中没找到，就会继续从后续的配置源查找，直到找到为止。)

例如，在给定此配置文件的情况下，`datastore.metric.host`和`datastore.metric.port`均已定义（并且可以被覆盖）。如果另外在默认值中定义了`datastore.metric.protocol`，Viper也会找到它。

然而，如果`datastore.metric`被直接赋值覆盖（被flag，环境变量，`set()`方法等等…），那么`datastore.metric`的所有子键都将变为未定义状态，它们被高优先级配置级别“遮蔽”（shadowed）了

###### 2.提取子树

还是以上面的配置文件为例：

~~~json
{
    "host": {
        "address": "localhost",
        "port": 5799
    },
    "datastore": {
        "metric": {
            "host": "127.0.0.1",
            "port": 3099
        },
        "warehouse": {
            "host": "198.0.0.1",
            "port": 2112
        }
    }
}
~~~

提取操作：

~~~go
	viper.SetConfigFile("指定配置文件路径")
	viper.ReadInConfig()
	value := viper.Sub("datastore.metric")
~~~

value此时代表：

~~~json
      "host": "127.0.0.1",
      "port": 3099
~~~

###### 3.反序列化

将所有或特定的值解析到结构体、map等。

方法(也可以使用函数，但是函数底层还是调用方法)：

~~~go
func (v *Viper) Unmarshal(rawVal any, opts ...DecoderConfigOption) error {
	return decode(v.AllSettings(), defaultDecoderConfig(rawVal, opts...))
}
func (v *Viper) UnmarshalKey(key string, rawVal any, opts ...DecoderConfigOption) error {
	return decode(v.Get(key), defaultDecoderConfig(rawVal, opts...))
}
~~~

官方案例：

~~~go
type config struct {
	Port int
	Name string
	PathMap string `mapstructure:"path_map"`
}

var C config

err := viper.Unmarshal(&C)
if err != nil {
	t.Fatalf("unable to decode into struct, %v", err)
}

~~~

如果你想要解析那些键本身就包含`.`(默认的键分隔符）的配置，你需要修改分隔符：

~~~go
v := viper.NewWithOptions(viper.KeyDelimiter("::"))
//自定义viper的配置，这里修改分隔符未::
v.SetDefault("chart::values", map[string]interface{}{
    "ingress": map[string]interface{}{
        "annotations": map[string]interface{}{
            "traefik.frontend.rule.type":                 "PathPrefix",
            "traefik.ingress.kubernetes.io/ssl-redirect": "true",
        },
    },
})

type config struct {
	Chart struct{
        Values map[string]interface{}
    }
}

var C config

v.Unmarshal(&C)

~~~

Viper还支持解析到嵌入的结构体：

~~~go
/*
Example config:

module:
    enabled: true
    token: 89h3f98hbwf987h3f98wenf89ehf
*/
type config struct {
	Module struct {
		Enabled bool

		moduleConfig `mapstructure:",squash"`
	}
}

// moduleConfig could be in a module specific package
type moduleConfig struct {
	Token string
}

var C config

err := viper.Unmarshal(&C)
if err != nil {
	t.Fatalf("unable to decode into struct, %v", err)
}

~~~

Viper在后台使用[github.com/mitchellh/mapstructure](https://github.com/mitchellh/mapstructure)来解析值，其默认情况下使用`mapstructure`tag。

**注意** 当我们需要将viper读取的配置反序列到我们定义的结构体变量中时，一定要使用`mapstructure`tag哦！

###### 4.序列化成字符串

通过AllSettions()方法，将所有设置键值对以map的形式返回

##### 八.使用单个viper实例

以上例子直接调用函数，都是viper的单例模式，其底层则是已经声明了一个viper对象的全局变量，且已经用New()函数初始化，函数里面都是该viper对象调用对应的方法。

这些函数都是viper单例模式的封装，使用户更加方便地使用viper。

##### 九.使用多个viper实例

~~~go
//使用New函数进行初始化一个viper对象
x := viper.New()
y := viper.New()

x.SetDefault("ContentDir", "content")
y.SetDefault("ContentDir", "foobar")
//...

~~~

##### 十.使用viper配置mysql示例：

配置文件:config.yaml

~~~yaml
mysql:
  name: root
  password: 666666
  ip: 127.0.0.1
  port: 3306
  dbname: example
~~~

连接数据库：

~~~go
package main

import (
	"fmt"
	"github.com/fsnotify/fsnotify"
	"github.com/spf13/viper"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
	"strings"
)

type Info struct {
	MysqlInfo `mapstructure:"mysql"`
}
type MysqlInfo struct {
	Name     string
	Password string
	Ip       string
	Port     string
	Dbname   string
}
type test struct {
	Name string `gorm:"column:name"`
}

func main() {
	var config = new(Info)
	viper.SetConfigFile("./config.yaml") //指定配置文件路径
	err := viper.ReadInConfig()          //读取配置信息
	if err != nil {
		panic(err)
	}
	viper.GetString("")
	//将配置信息保存到结构体中
	err = viper.Unmarshal(config)
	if err != nil {
		panic(err)
	}
	//连接数据库操作
	dsn := strings.Join([]string{config.Name, ":", config.Password, "@tcp(", config.Ip, ":", config.Port, ")/", config.Dbname, "?charset=utf8mb4&parseTime=True&loc=Local"}, "")
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
	if err != nil {
		panic(err)
	}
	db.AutoMigrate(&test{})//同步测试表
	//监控配置文件变化
	viper.WatchConfig()
	// 注意！！！配置文件发生变化后要同步到全局变量
	viper.OnConfigChange(func(in fsnotify.Event) {
		fmt.Println("夭寿啦~配置文件被人修改啦...")
		if err := viper.Unmarshal(config); err != nil {
			panic(fmt.Errorf("unmarshal conf failed, err:%s \n", err))
		}
	})
	//...
}

~~~


