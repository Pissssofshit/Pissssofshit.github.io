---
title: go-blog系列(2) 参数校验
date: 2020-11-05 14:49:32
tags:
---



gin整合了校验参数的库

```
https://github.com/go-playground/validator
```

#### 基本用法介绍

```
type Param struct {
	param1 time.Time `form:"param_form" json:"param_json" 	       uri:"param_uri" binding:"tag1,tag2"`
}
```

form、json、uri分别代表从表单、json、uri中获取参数值

binding标识具体的验证方法

##### binding可选的验证方法举例

###### 跨字段验证

即与其他字段比较值进行验证，如

```
- eqfield 
	与另一字段相同
	用法:
		type Param struct {
			Param1 string `json:"parma1" binding:"eqfiled=Param2"`
 			Param2 string `json:"param2" binding:"required"`
		}
		则Param.Param1字段需与Param.Param2字段相同
- nefield
	与另一字段不同
- gtfield
- gtefield
- ltfield
- ltefield
```

这些tag有相应的cs版本，如下

```
- eqcsfield
- necsfield
- gtcsfield
- gtecsfield
- ltcsfield
- ltecsfield
```

两者的不同在于后者能用于检测不同层级之间的字段值，示例如下:

```
type Inner struct {
	StartDate time.Time
}

type Outer struct {
	InnerStructField *Inner
	CreatedAt time.Time      `binding:"ltecsfield=InnerStructField.StartDate"`
}
```

###### 字段值验证

即检测自身的值、格式是否符合条件，如

```
email
numberic
max=10
等
```

###### 条件验证

即在特定的情况下才需要验证的字段，如

```
required_if //当某字段为空时，此字段不为空
required_unless //只有某字段为空时，此字段不为空
等，不一一列举
```

还有一个比较重要的是

```
omitempty //当字段值为空值时，不进行验证，此tag与上面的tag进行组合之后，可以定制出多种规则，后面的代码讲解中会用到
```

###### 自定义tag

代码示例如下

```
func NewUserService() UserService {
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
		v.RegisterValidation("custom_tag", custom_tag) // 注册自定义tag
	}
	return UserService{}
}
//自定义tag函数
var custom_tag validator.Func = func(fl validator.FieldLevel) bool {
	phone, ok := fl.Field().Interface().(string)
	if ok {
		regular := "^((13[0-9])|(14[5,7])|(15[0-3,5-9])|(17[0,3,5-8])|(18[0-9])|166|198|199|(147))\\d{8}$"
		reg := regexp.MustCompile(regular)
		return reg.MatchString(phone)
	}
	return false
}
```

注册的custom_tag就可以像required之类的自带tag一样使用了

###### 验证方法

验证方法分为两大类

- Bind

- ShouldBind

  前者在遇到参数错误时直接退出并返回**c.AbortWithError(400, err).SetType(ErrorTypeBind)** 

  后者会返回错误由用户处理

  并且各自都有衍生方法标明参数来源，如BindJson,BindXml等，示例如下

```
	route.GET("/:name/:id", func(c *gin.Context) {
		var person Person
		if err := c.ShouldBindUri(&person); err != nil {
			c.JSON(400, gin.H{"msg": err})
			return
		}
		c.JSON(200, gin.H{"name": person.Name, "uuid": person.ID})
	})
```



#### blog代码实例

首先是注册接口，定义的参数结构体是

```
type Create struct {
	Username string `json:"user_name" binding:"required"` //必须字段
	Phone    string `json:"phone" binding:"required_without=Email,omitempty,numeric,custom_tag"` //phone和email作为联系方式二选一,不能都为空,在有值时验证格式
	Email    string `json:"email" binding:"required_without=Phone,omitempty,email"`
	Password string `json:"pass_word" binding:"required"`
	Sex      string `json:"sex" binding:"required,oneof=man woman"`//性别在男、女中二选一
}
```

Phone和Email字段的校验tag做下解释:

有多个校验器存在时，校验顺序为从左往右

required_without 当另一值为空时不能为空

omitempty 在值为空时放弃检测(说明另一个值有值并且这个值没有填，不需要检测)

custom_tag 是自定义的tag,代码再自定义的校验器处已经给出，用于检测手机号格式



如果参数同时存在于uri和post中，使用shouldBindUri和shouldBindJson分开校验，如下

```
//bind uri
type EditUri struct {
	UserId string `uri:"user_id" binding:"required"`
}
type EditJson struct {
	Phone    string    `json:"phone" binding:"omitempty,numeric"`
	Email    string    `json:"email" binding:"omitempty,email"`
	Password string    `json:"pass_word" binding:"-"`
	Birthday time.Time `json:"birthday" binding:"omitempty,lte"`
	Sex      string    `json:"sex" binding:"omitempty,oneof=man woman"`
}

func (userService *UserService) Edit(c *gin.Context) {
	var json EditJson
	var uri EditUri
	if err := c.ShouldBindUri(&uri); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	if err := c.ShouldBindJSON(&json); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	c.JSON(http.StatusOK, gin.H{"status": "success"})
}

```

完整代码查看我的github项目

    git clone git clone https://github.com/Pissssofshit/go-blog

,并检出分支

    git checkout 参数校验

在项目下运行

    go run main.go

项目就会运行在8080端口下