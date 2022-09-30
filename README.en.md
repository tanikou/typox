a tiny and simple entity mapper for frontend. inspired by typeorm

`extendable，simple config，support typescript, zero dependency`

# install

```
npm i --save typox
```

# define entity

```
import { Entity, Model, type } from 'typox'

@Entity()
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

example 1: parse data from backend api

```
new Named().parse({ name: 'Typox', date: new Date(), image: 'x' })
```

get result

```
Named {name: "Typox", avatar: "x"}
```

example 2: new object for frontend data

```
new Named({ name: 'Typox', avatar: 'x' })
or
new Named().merge({ name: 'Typox', avatar: 'x' })
```

get result

```
Named {name: "Typox", avatar: "x"}
```

example 3: `attr not match`

```
new Named().parse({ date: new Date() })
```

get result

```
Error: Named.name defined as String, got：undefined
```

# customize error message

```
import { setMessageFormat } from 'typox'

setMessageFormat('{entity}.{attr}: {value} is not "{type}"')
```

get result

```
Error: Named.name: undefined is not "String"
```

# not match handler.

by default, it will throw an error if not match.

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

# `extend` others and more `anotations`

```
import { Entity, Model, type, from, nullable, parse, validator } from 'typox'

@Entity()
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

new Staff().parse({
  name: 'Typox Mapper',
  author: { name: 'tan' },
  birthday: new Date()
})
```

```
Staff { name: 'Typox Mapper', author: 'tan', email: '', year: 2021 }
```

# as props type of vue component

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

# mult type and enumeration

```
import { Entity, Model, type, enumeration } from 'typox'

@Entity()
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

good

```
new Named().parse({ name: 'Typox', key: 1,  age: 10})

get

Named { name: "Typox", key: 1,  age: 10 }


new Named().parse({ name: 'Typox', key: 'private key',  age: 10})

get

Named { name: "Typox", key: "private key",  age: 10 }
```

error

```
new Named().parse({ name: 'Typox', key: 1,  age: 30})

get

Error: Named.age defined as [10, 20], 30
```

# reverse entity to json object for api request

```
import { Entity, Model, type, to, reverse } from 'typox'

@Entity()
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

get

Query { name: "typox", key: "123", addr: '' }


entity.reverse()

get

{ "userName": "typox", "privatekey": "base64://123" }
```

entity.reverse() will ingore `null`, `''`, `undefined` property by default, you can use `entity.reverse({ lightly: false })` to get `{ "userName": "typox", "privatekey": "base64://123", "addr": "" }`

```
interface ReverseOption {
  // lightly mode will ignore attribute if the value is blank string
  lightly?: boolean | undefined
  // ignore attribute
  exclusion?: string[]
}
```

# set lightly for global

```
import { defaults } from 'typox'

defaults.lightly = false
```

# decorators

1. @`Enitity` or @`entity` => anotation for entity class
2. @`from` => define from original prop. eg: `@from('company.name')`. or `@from(['name', 'member.name'])`
3. @`type` => define data type. support mult type. eg: `@type([Number, String])`
4. @`nullable` => allow data can be `null` or `undefined`
5. @`parse` => define how to parse the original data. eg: `@parse((v, me) => (v * me.unit) + 'miniute')`. `this` keyword in parse function is supported from V2.8.1
6. @`enumeration` => enumeration
7. @`validator` => support mult validator
8. @`omit` => omit this attribute when call functioin parse, merge, reverse
9. @`to` => define how to format entity to a json object, generally for the backend api. eg:tranfer entity attr`name` to `userName`，`@to('userName')`
10. @`reverse` => define how to format the entity attr to json attr.
    eg: `@reverse((v, me) => me.status === 1 ? moment(v).format('YYYYMMDD') : moment(v).format('YY-MM-DD'))`
    or : `@reverse(function(v) { return this.status === 1 ? v : '' })`
11. @`recover` => define how to convert string vlaue to the attritube value from query
12. `@format` alias to `@parse`
13. `@decorators` => define custom decorators.
14. `@dep` => define attribute dependencies, typox will parse dependencies before this attribute
15. @`merge` => define how to copy data. such as array to object. eg: `@merge((v, me) => new Staff(v)`.

# custom decorators

```
import { Entity, Model, type, decorators } from 'typox'

@Entity()
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

run:

```
new Tm().runDecorators('max')
```

get result

```
{name: 5, id: 0}
```

# private function of entity from model

1. `entity.parse`(source, option?: ParseOption), the parameter `source` can be a json object or a entity, normally use this function to merge some values from other object
2. `entity.merge`(source), the parameter `source` can be a json object or a entity, this will set the attribute value of entity by source attr value (if the source has the same attribute)
3. `entity.recover`(source), the parameter `source` can be a json object or a entity, normally use this function to recover entity from the url. this function will auto tranform prop data to the type of defined. eg: http://localhost/logs?name=typox&page=1&size=10. create entity by `new LogQuery().recover({ name: 'typox', page: '1', size: '10' })` get result `LogQuery { name: 'typox', page: 1, size: 10 }`
4. `entity.reverse()` transform entity to a json object
5. `entity.runDecorators(attr: string)` run custom decorator

```
interface ParseOption {
  // nullable for all attribute
  nullable?: boolean
  // ignore validator
  validate?: boolean
}
```

# more

you can define some prop without anotation. it will ignore when create a entity or reverse to a json object. eg:

```
@Entity()
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

get

```
Named { loading: false, name: "typox" }
```

`named.isme()` will get `true`

`named.reverse()` will get json object `{ name: 'typox' }`

especially useful for data lock in table

# use this in parse function

by default, typox will parse attributes one by on as you defined.

```
@Entity()
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

output:
1
2 1
```

you can define @parse by normal function as `@parse(function () { return this.min + this.max })`, but in TypeScript you will get an eslint error `'this' implicitly has type 'any' because it does not have a type annotation.`, you can resovle this by `@parse(function (this: Staff, v) { return this.min + v })`

# method parameter check in runtime

if your project do not support typescript, but you want to check the parameter in runtime

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
