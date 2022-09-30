简单好用的前端实体映射工具

`Entity可继承，简单配置，可直接用作vue组件prop的类型限制。支持TS。零依赖`

1. 强化数据结构及类型
2. 强化组件参数类型
3. 快速与后端接口数据属性对应(后端到前端，前端到后端)及应对字段名变更
4. 强化数据校验，在开发阶段提前发现数据问题或代码处理异常
5. 强化接口参数校验，提前发现函数调用时异常
6. 类支持继承复用，可扩展方法，减少重复代码
7. 分离数据转换处理，使代码其他位置只关心业务逻辑，不再包含数据转换逻辑
8. 快速还原恢复数据（如从 URL 中恢复成业务数据）

# 安装

```
npm i --save typox
```

# 定义实体类

```
import { entity, Model, type } from 'typox'

@entity()
export default class Named extends Model {
  constructor (source) {
    super()
    this.merge(source)
  }

  @type(String)
  name = ''

  @from('image')
  @type(String)
  @nullable()
  avatar = ''
}
```

例 1:转换后端数据（属性key不完全一致）

```
new Named().parse({ name: 'Typox', date: new Date(), image: 'x' })
```

结果:

```
Named {name: "Typox"}
```

例 2: 使用前端数据（保留属性完全匹配的数据）

```
new Named({ name: 'Typox', avatar: 'x' })
或者
new Named().merge({ name: 'Typox', avatar: 'x' })
```

结果:

```
Named {name: "Typox", avatar: "x"}
```

例 3: `属性与定义不匹配的情况`

```
new Named().parse({ date: new Date() })
```

运行后抛出一个 Error。（可通过`setLogger`自定义是抛出错误还是弹窗提示）

```
Error: Named.name defined as String, got：undefined
```

# 自定义错误信息

```
import { setMessageFormat } from 'typox'

setMessageFormat('{entity}.{attr}: {value} is not "{type}"')
```

运行结果

```
Error: Named.name: undefined is not "String"
```

# 自定义不匹配时的处理

在类型不匹配时默认会抛出一个 Error，你可以设置 logger 覆盖默认处理

```
import { setLogger } from 'typox'

setLogger(console)
```

or

```
setLogger({
  error: (v: string) => { alert(v) }
})
```

# `继承` 其他实体类 及 更多 `注解`

```
import { entity, Model, type, from, nullable, parse, validator } from 'typox'

@entity()
export default class Staff extends Named {
  constructor (source) {
    super()
    this.merge(source)
  }

  @from('author.name')
  @type(String)
  author = ''

  @from('author.email')
  @nullable()
  email = ''

  @from('birthday')
  @type(Number)
  @parse(v => v.getFullYear())
  @validator([
    (v) => v.getFullYear() > 2020
  ])
  year = ''
}

// 通过映射关系赋值
new Staff().parse({
  name: 'Typox Mapper',
  author: { name: 'tan' },
  birthday: new Date()
})

or

// 完全通过实体属性名赋值
new Staff({
  name: 'Typox Mapper',
  author: 'tan',
  year: 2021
})
```

get

```
Staff { name: 'Typox Mapper', author: 'tan', email: '', year: 2021 }
```

# 实体类可以直接做为 vue 的 prop type，强化组件的数据类型

```
<template>
  <div>
    {{ staff }}
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'
import Staff from '@/entity/Staff'

export default defineComponent({
  props: {
    staff: {
      type: Staff
    }
  }
})
</script>
```

# 多数据组件及枚举支持

```
import { entity, Model, type, enumeration } from 'typox'

@entity()
export default class Named extends Model {
  constructor (source) {
    super()
    this.merge(source)
  }

  @type(String)
  name = ''

  @type([String, Number])
  key = ''

  @enumeration([10, 20])
  age = 0
}
```

正常的

```
new Named().parse({ name: 'Typox', key: 1,  age: 10})

生成

Named { name: "Typox", key: 1,  age: 10 }


new Named().parse({ name: 'Typox', key: 'private key',  age: 10})

生成

Named { name: "Typox", key: "private key",  age: 10 }
```

异常的值（30）

```
new Named().parse({ name: 'Typox', key: 1,  age: 30})

报错

Error: Named.age defined as [10, 20], 30
```

# 将前端的实体类转换成后端 API 所需要的 json object

```
import { entity, Model, type, to, reverse } from 'typox'

@entity()
export default class Query extends Model {
  constructor (source) {
    super()
    this.merge(source)
  }

  @from('nickname')
  @type(String)
  @to('userName')
  name = ''

  @type(String)
  @to("privatekey")
  @reverse((v) => 'base64://' + v)
  key = ''

  @to()
  addr = ''
}
```

```
const entity = new Query().parse({ nickname: 'typox', key: '123' })

生成类

Query { name: "typox", key: "123", addr: '' }


entity.reverse()

转换成json为

{ "userName": "typox", "privatekey": "base64://123" }
```

entity.reverse() 默认将忽略属性值为 `null`, `''`, `undefined`及未定义 to 的属性,不传值给后端, 可以使用 `entity.reverse({ lightly: false })` 会将所有定义了 to 的属性都传回去 `{ "userName": "typox", "privatekey": "base64://123", "addr": "" }`

reverse 参数

```
interface ReverseOption {
  // 轻便模式下会忽略值为空字符串的属性不生成到json当中
  lightly?: boolean | undefined
  // 强制忽略某些属性到json中，效果等同于配置了omit
  exclusion?: string[]
}
```

# 全局设置 lightly

```
import { defaults } from 'typox'

defaults.lightly = false
```

# 其他装饰器

1. @`Enitity` or @`entity` => 注解在类上
2. @`from` => 定义字段数据来源，可多级结构。例`@from('company.name')`，或多属性来源，`@from(['name', 'member.name'])`
3. @`type` => 定义字段数据类型，可是基础数据类型也可以类，可设置多类型,例: `@type([Number, String])`
4. @`nullable` => 设置是否允许为空，即允许值为`null`或`undefined`
5. @`parse` => 用于自定义格式化转换数据。例：`@parse((v, me) => (v * me.unit) + '分钟')`。从 V2.8.1 开始支持优化转换通过`this`引用到的其他属性。也可以通过箭头函数使用第二个参数拿到原始数据自己处理。
6. @`enumeration` => 设置数据只能是枚举的值
7. @`validator` => 自定义校验
8. @`omit` => 在 parse 或者 merge 或者 reverse 时忽略此属性
9. @`to` => 定义将属性名转成换其他属性名，一般用于转给后端接口。例:类属性`name`转换成`userName`，`@to('userName')`
10. @`reverse` => 自定义在 to 时如何转换属性。
    例: `@reverse((v, me) => me.status === 1 ? moment(v).format('YYYYMMDD') : moment(v).format('YY-MM-DD'))` 或者使用 this 引用其他属性：`@reverse(function(v) { return this.status === 1 ? v : '' })`
11. @`recover` => 定义如何将字符串值转换为属性值，如`@recover(Date)`或`@recover(v => new Date(Number(v)))`，例从 query 中`date[]=1655827200000&date[]=1656086400000`还原为`date: [new Date(1655827200000), new Date(1656086400000)]`。如果有指定`recover`则使用指定的类型转换，如果没有配置类型则用配置的类型进行初始化，没有配置则默认用属性的默认值的类型进行初始化，基础数据类型均可自动转换
12. @`format` => 等价于@`parse`
13. `@decorators` => 定义自定义装饰器规则
14. `@dep` => 用于标记此属性需要依赖哪些属性，要求先解析依赖的属性
15. @`merge` => 定义merge时如何复制数据, 用于复杂数据类型自定义复制. eg: `@merge((v, me) => new Staff(v)`。如果没有设置@merge则判断是为数据进行map否则直接赋值

# 自定义装饰

```
import { entity, Model, type, decorators } from 'typox'

@entity()
export class Tm extends Model {
  constructor (source?: Record<string, unknown>) {
    super()
    this.merge(source)
  }

  @decorators({
    max: () => 0
  })
  @type(Number)
  id = 0

  @decorators({
    max: function (this: Tm) {
      return this.id + 5
    }
  })
  @type(String)
  name = ''
}
```

执行:

```
new Tm().runDecorators('max')
```

结果：

```
{name: 5, id: 0}
```

# 从 Model 基类继续到的私有方法

1. `entity.parse`(source, option?: ParseOption), 参数 `source` 可能是 json 也可以是实体类, 通常用来合并其他地方的属性值到实体中。一般用于将后端数据通过映射关键转换成实体类
2. `entity.merge`(source), 参数 `source` 可能是 json 也可以是实体类, 用于其他地方的具有相同属性值覆盖到实体中，如表格组件提交的新的分页或排序数据。一般用于前端自己构造或覆盖属性值
3. `entity.recover`(source), 通用用来将 url 中的参数还原给实体类. `recover` 将自动识别`type`定义的类型并转换成指定的类型. 如: http://localhost/logs?name=typox&page=1&size=10. 用 URL 中的参数合并到实体 `new LogQuery().recover({ name: 'typox', page: '1', size: '10' })` 得到结果 `LogQuery { name: 'typox', page: 1, size: 10 }`
4. `entity.reverse()` 将实体转换为 json 数据，一般用于提交给后端
5. `entity.runDecorators(attr: string)` 执行自定义装饰器并得结果

```
interface ParseOption {
  // 本次转换是否全部属性允许空
  nullable?: boolean
  // 本次转换是否忽略validator校验处理
  validate?: boolean
}
```

# 其他

你可以定义没有任务注解的属性或者方法。这时在生成实体与实体转换成 json 将忽略这些无任务注解的属性。 eg:

```
@entity()
export default class Named extends Model {
  constructor (source) {
    super()
    this.merge(source)
  }

  loading = false

  isme () {
    return this.name === 'typox'
  }

  @type(String)
  name = ''
}
```

```
new Named().parse({ loading: true, name: 'typox' })
```

生成类

```
Named { loading: false, name: "typox" }
```

使用自定义的 `named.isme()` 获得结果 `true`

使用从 model 继承的方法 `named.reverse()` 生成 json `{ name: 'typox' }`

一般在表格的行操作需要加防抖时特别好用

# 在 parse 中使用 this

默认情况下将会按你定义的属性从上向下一个一个的进行转换属性值。但如果你在某个方法中使用了 this.xxx，则将会优化转换你引用到的 xxx 属性。注意不要相互交叉引用否则可能会出问题

```
@entity()
export class Staff extends Model {
  constructor (source?: Record<string, unknown>) {
    super()
    this.merge(source)
  }

  @parse(function (this: Staff, v) {
    console.log(v, this.id)
    return v
  })
  @nullable()
  @type(String)
  @to()
  key = ''

  @parse((v) => {
    console.log(v)
    return v
  })
  @type(Number)
  @to()
  id = 0
}

new Staff().parse({ id: 1, key: '2' })

输出:
1
2 1
```

通常你可以用普通 function 定义 `@parse(function () { return this.min + this.max })`, 但是在 ts 中会有 any 报错提示： `'this' implicitly has type 'any' because it does not have a type annotation.`, 你可以用变量 this 去解决这个问题： `@parse(function (this: Staff, v) { return this.min + v })`。在类定义这使用箭头函数时 this 将永远不会是你想要的实体 this。
在部分情况下因为编译不能自动识别属性依赖关系可使用`@dep(['id'])`明确依赖关系

# 方法执行时参数校验

如果你项目中没有用到 TS，又需要在开发阶段发时方法执行时获得的参数错误情形，即更多的发现出错的可能，可以使用@param 进行注解

```
import { param } from 'typox'
import { Log } from '../entity/log'

export class LogRepository extends Repository {
  @param(Log)
  static async create (row) {
    return axios.post('/system/log', row)
  }
}
```

```
LogRepository.create(new Member())
```

get

```
Error: LogRepository.create(Member <> Log)
```

此错误信息也可以通过设置`setLogger`处理异常信息
