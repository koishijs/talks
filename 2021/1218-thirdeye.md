---
theme: seriph
background: https://minio.momobako.com:9000/nanahira/koishi-talk/20211218-thirdeye.jpg
class: text-center
highlighter: shiki
---

# Koishi Talk - External

<div class="opacity-80">
18.5th / 2021-12-18
</div>

---

# Koishi Thirdeye

* 装饰器 / DI 风格的 Koishi 插件框架。

* 基于 `koishi-nestjs` 的风格重构而成。

```ts
@KoishiPlugin()
export default class MyPlugin {
  constructor(ctx: Context, config: Config) {}

  @UseCommand('echo <content:string>')
  onEcho(@PutArg(0) content: string) {
    return content;
  }
}
```

---


# 历史

### Koishi v3

```ts
export const name = 'my-plugin';
export function apply(ctx: Context, config: Config) {
  ctx.command('echo <content:string>')
    .action((argv, content) => content);
}
```

### Koishi v4 alpha


```ts
export const name = 'my-plugin';
export const schema = Schema.object({
  foo: Schema.string().default('baz'),
  bar: Schema.number().required(),
});

export function apply(ctx: Context, _config: Config) {
  const config = Schema.validate(schema, _config);
  ctx.command('.echo <content:string>')
    .action((argv, content) => content);
}
```

---

### Koishi v4 beta

```ts
export default class MyPlugin {
  static schema = Schema.object({
    foo: Schema.string().default('baz'),
    bar: Schema.number().required(),
  });
  constructor(private ctx: Context, private config: Config) {
    ctx.command('echo <content:string>')
      .action((argv, content) => content);
  }
}
```

### koishi-nestjs

```ts
// in Nest.js
@Injectable()
export class AppService {
  @UseCommand('echo <content:string>')
  onEcho(@PutArg(0) content: string) {
    return content;
  }
}
```

---

## Koishi Thirdeye

```ts
@DefineSchema()
export class Config {
  constructor(_config: any) {}

  @SchemaProperty({ default: 'baz' })
  foo: string;

  @SchemaProperty({ required: true })
  bar: number;
}

@KoishiPlugin({ name: 'my-plugin', schema: Config })
export default class MyPlugin {
  constructor(ctx: Context, config: Partial<Config>) {}

  @UseCommand('echo <content:string>')
  onEcho(@PutArg(0) content: string) {
    return content;
  }
}
```

---

# 定义插件

Koishi Thirdeye 的插件必须是类插件，且使用 `@KoishiPlugin()` 注解。

```ts
// 在此处定义 Config 的 Schema 描述模式

@KoishiPlugin({ name: 'my-plugin', schema: Config })
export default class MyPlugin {
  constructor(ctx: Context, config: Partial<Config>) {} // 不建议在构造函数进行任何操作
}
```

---

# 属性注入

下列各属性装饰器会在**构造函数运行后**对成员变量进行注入。

> 请不要在构造函数中进行对这些字段对访问。

```ts
@KoishiPlugin({ name: 'my-plugin', schema: Config })
export default class MyPlugin {
  constructor(ctx: Context, config: Partial<Config>) {}
  
  @InjectContext()
  private ctx: Context; // 建议如此使用 Context，而不是构造函数中的

  @InjectConfig()
  private config: Config; // 建议如此使用 Config，而不是构造函数中的

  @InjectLogger('my-plugin') // Logger 名称默认为插件名称
  private logger: Logger;

  @Inject('cache', true)
  private cache: Cache; // 注入 Service API 中的 Cache，并加入 using 列表

  @Inject()
  private database: Database; // 根据属性名称判别 Service API 名称
}
```

---

# 钩子方法

钩子方法会在特定的时机被调用。

```ts
@KoishiPlugin({ name: 'my-plugin', schema: Config })
export default class MyPlugin implements OnApply, OnConnect, OnDisconnect { // 实现对应的钩子方法即可
  constructor(ctx: Context, config: Partial<Config>) {} // 不建议在构造函数进行任何操作
  
  onApply() {} // 会在插件被加载时立即运行，只可以同步

  async onConnect() {} // 会在 Koishi 启动时运行，可以异步，等价于 ctx.on('connect', async () => {})

  async onDisconnect() {} // 会在插件被卸载时运行，可以异步，等价于 ctx.on('disconnect', async () => {})
}
```

---

# schemastery-gen

类装饰器定义的 Schema 模式。

* 类将会自动实例化，并注入这些方法。

* 使用 `type` 参数手动指定类型或另一个 Schema 对象。不指定则会自动推断。

```ts
@DefineSchema() // Config 类本身会成为 Schema 对象
export class Config {
  constructor(_config: any) {}

  @SchemaProperty({ default: 'baz' })
  foo: string; // 自动推断出 Schema.string()

  getFoo() {
    return this.foo;
  }

  @SchemaProperty({ type: Schema.number(), required: true }) // 也可手动指定 Schema 对象
  bar: number;

  @SchemaProperty({ type: String })
  someArray: string[]; // 自动推断出 Schema.array(...)，但是无法推断内部类型，需要手动指定。
}
```

---

# 嵌套 Schema

嵌套的情况下，内层类也会自动实例化，并且会自动注入到外层类中。

```ts
@DefineSchema()
export class ChildConfig {
  constructor(_config: any) {}

  @SchemaProperty({ default: 'baz' })
  foo: string;

  @SchemaProperty({ type: Schema.number(), required: true })
  bar: number;
}

@DefineSchema() // Config 类本身会成为 Schema 对象
export class Config {
  constructor(_config: any) {}

  @SchemaProperty()
  child: ChildConfig; // 自动推断出 ChildConfig

  @SchemaProperty({ type: ChildConfig })
  children: ChildConfig[]; // 无法自动推断 ChildConfig，需要手动指定。但是可以推断出外层的 Schema.array(...)
}

const Config = Schema.array(ChildConfig) // NG: 无法自动实例化 ChildConfig 类
```

---

# 注册事件

```ts
@KoishiPlugin({ name: 'my-plugin', schema: Config })
export default class MyPlugin {
  constructor(ctx: Context, config: Partial<Config>) {}
  
  @UseMiddleware() // 等价于 ctx.middleware
  someMiddleware(session: Session, next: NextFunction) {
    return next();
  }

  @UseEvent('message') // 等价于 ctx.on
  onMessage(session: Session.Payload<'message'>) {
    return;
  }

  @Get('/ping') // 等价于 ctx.router.get
  async ping(ctx: KoaContext) {
    ctx.body = 'pong';
  }
}
```

* `@UseBeforeEvent` 对应 `ctx.before`。

* `@Post` 以及其余的 HTTP 方法。

---

# 注册指令

```ts
@KoishiPlugin({ name: 'my-plugin', schema: Config })
export default class MyPlugin {
  constructor(ctx: Context, config: Partial<Config>) {}
  
  @UseCommand('echo', '又是一个 echo 命令')
  @CommandUsage('Blah blah blah')
  onEcho(
    @PutOption('content', '-c <content:string>  内容') content: string
  ) {
    return content;
  }
}
```

### Command 定义装饰器

* `@CommandUsage(text: string)` 指令介绍。等价于 `cmd.usage(text)`。

* `@CommandExample(text: string)` 指令示例。等价于 `cmd.example(text)`。

* `@CommandAlias(def: string)` 指令别名。等价于 `cmd.alias(def)`。

* `@CommandShortcut(def: string, config?)` 指令快捷方式。等价于 `cmd.shortcut(def, config)`。

---

### Command 命令参数装饰器

* `@PutArgv()` 注入 `Argv` 对象。

* `@PutSession(field?: keyof Session)` 注入 `Session` 对象，或 `Session` 对象的指定字段。

* `@PutArg(index: number)` 注入指令的第 n 个参数。

* `@PutOption(name: string, desc: string, config: Argv.OptionConfig = {})` 给指令添加选项并注入到该参数。等价于 `cmd.option(name, desc, config)` 。

* `@PutUser(fields: string[])` 添加一部分字段用于观测，并将 User 对象注入到该参数。

* `@PutChannel(fields: string[])` 添加一部分字段用于观测，并将 Channel 对象注入到该参数。

* `@PutUserName(useDatabase: boolean = true)` 注入当前用户的用户名。
  * `useDatabase` 是否尝试从数据库获取用户名。

---

# 子指令

Koishi Thirdeye 中，子指令需要用完整的名称进行声明。

* 对于没有回调的父指令，可以使用 `empty` 选项，使其不具有 action 字段。

```ts
@KoishiPlugin({ name: 'my-plugin', schema: Config })
export default class MyPlugin {
  constructor(ctx: Context, config: Partial<Config>) {}
  
  @UseCommand('ygopro', 'YGOPro 相关指令', { empty: true })
  ygoproCommand() {
    // 该命令不会有 action，因此该方法不会被调用。
  }

  @UseCommand('ygopro.rank', 'YGOPro 排名')
  @CommandUsage('查询玩家的 YGOPro 排名')
  ygoproRankCommand(@PutOption('player', '-p <name:string>  玩家名称') player: string) {

  }

  @UseCommand('ygopro.rooms', 'YGOPro 房间')
  @CommandUsage('查询 YGOPro 房间列表')
  ygoproRoomsCommand() {

  }
}
```

---

# 嵌套插件与异步插件

```ts
import PluginCommon from '@koishijs/plugin-common';

@KoishiPlugin({ name: 'my-plugin', schema: Config })
export default class MyPlugin {
  constructor(ctx: Context, config: Partial<Config>) {}
  
  @UsePlugin()
  registerPluginCommon() { // 会于插件注册时立即运行，并取返回值作为插件的嵌套插件
    return PluginDef(PluginCommon, { echo: true })
  }

  private async getPluginCommonConfig() {
    return { echo: true };
  }

  @UsePlugin()
  async registerAsyncPluginCommon() { // 可以是异步插件
    const pluginCommonConfig = await this.getPluginCommonConfig();
    return PluginDef(PluginCommon, pluginCommonConfig);
  }
}
```

---

# 作用域

作用域装饰器可以用来指定该插件类总体，以及部分注册时间，指令，或嵌套插件的作用域。

* 作用域装饰器对插件类本身，以及中间件，事件，指令，嵌套插件，都有效。

* 使用 `@InjectContext()` 注入的 Context 对象，会受到该作用域的限制。

```ts
@OnPlatform('onebot')
@KoishiPlugin({ name: 'my-plugin', schema: Config })
export default class MyPlugin {
  constructor(ctx: Context, config: Partial<Config>) {}

  @InjectContext()
  private ctx: Context; // 该 Context 对象只对 OneBot 平台有效
  
  @OnGuild()
  @UseEvent('message') // 只对 OneBot 平台的群组有效
  onMessage(session: Session.Payload<'message'>) {
    return;
  }
}
```

---

# 作用域定义

* `@OnUser(value)` 等价于 `ctx.user(value)`。

* `@OnSelf(value)` 等价于 `ctx.self(value)`。

* `@OnGuild(value)` 等价于 `ctx.guild(value)`。

* `@OnChannel(value)` 等价于 `ctx.channel(value)`。

* `@OnPlatform(value)` 等价于 `ctx.platform(value)`。

* `@OnPrivate(value)` 等价于 `ctx.private(value)`。

* `@OnSelection(value)` 等价于 `ctx.select(value)`。

* `@OnContext((ctx: Context) => Context)` 手动指定上下文选择器，用于不支持的选择器。例如，

```ts
@OnContext(ctx => ctx.platform('onebot'))
```

---

# Service 提供者

Koishi Thirdeye 使用 `@Provide` 进行 Service API 提供者声明，提供依赖注入 (DI) 风格的 IoC 容器框架。

* 等价于 Koishi 的 Service 基类，但是在 Koishi Thirdeye 中采用了无侵入式的设计。

* `@Provide` 会自动完成 `Context.service(serviceName)` 的声明，但是仍要进行类型合并定义。

```ts
import { Provide, KoishiPlugin } from 'koishi-thirdeye';

// 类型合并定义不可省略
declare module 'koishi' {
  namespace Context {
    interface Services {
      myService: MyServicePlugin;
    }
  }
}

// `@Provide(name)` 装饰器会自动完成 `Context.service(name)` 的声明操作
@Provide('myService', { immediate: true })
@KoishiPlugin({ name: 'my-service' })
export class MyServicePlugin {
  // 该类会作为 Koishi 的 Service 供其他 Koishi 插件进行引用
}
```

---

# 模块化 / DI

```ts
@Provide('myModuleFoo')
@KoishiPlugin()
export class MyModuleFoo {
  @Inject(true)
  private database: Database;
}

@Provide('myModuleBar', { immediate: true })
@KoishiPlugin()
export class MyModuleBar {
  @Inject(true)
  private myModuleFoo: MyModuleFoo;
  @Inject(true)
  private cache: Cache;
}

@KoishiPlugin()
export default class MyPlugin {
  constructor(ctx: Context, config: Partial<Config>) {}
  @Inject(true)
  private myModuleFoo: MyModuleFoo;
  @Inject(true)
  private myModuleBar: MyModuleBar;
}
```

---

# Free Questions
