# Profiles

（译注：这个单词真的很难翻译，翻译成配置和configure重合，翻译成轮廓又莫名其妙，我的理解大概就是同一种配置的不同实例）

在项目中，总有一组需要基于运行的任务或者运行任务的人的角色的不变的选项。

在Erlang项目的例子中，最常见的例子是只在运行测试时所需要的依赖项，如模拟库或者框架特定的测试工具。

Rebar3通过Profiles的概念满足了此类需求。一个profile是一组只用于某个特定上下文的配置设置，覆盖或者补充了常规配置。目标是支持多种开发用例，保持事物的可重复性、不需额外工具或者环境变量来完成那些工作。

运行所用profile可通过3种不同方式来指定：

1. 调用`rebar3 as <profile> <command>`或者`rebar3 as <profile1>,<profile2> <command>`
2. 通过rebar3命令给定，比如，eunit和ct命令运行通常会添加test这个profile
3. 使用`REBAR_PROFILE`环境变量

这些形式中的任何一种（甚至一次性全部使用）都会让rebar3知道它应该作为一个特殊的profiles运行并相应修改它的配置。

profile配置能够在主rebar.config文件中如下指定：

```erlang
{profiles, [{ProfileName1, [Options, ...]},
            {ProfileName2, [Options, ...]}]}.
```

举个例子，一个测试profile添加了只在运行测试的时候使用的meck依赖项，被定义为：

```erlang
{profiles, [{test, [{deps, [meck]}]}]}. 
```

任何配置值都能配进profiles，包括插件、编译选项、发布选项等等。

## 例子

一个比较完善的例子可能看起来像这样：

```erlang
{deps, [...]}.
{relx, [
    ...
]}.

{profiles, [
    {prod, [
        {erl_opts, [no_debug_info, warnings_as_errors]},
        {relx, [{dev_mode, false}]}
    ]},
    {native, [
        {erl_opts, [{native, {hipe, o3}}]}
    ]},
    {test, [
        {deps, [meck]},
        {erl_opts, [debug_info]}
    ]}
]}.
```

这样的项目有4个不同的profiles：

1. 默认（default），对应整个rebar.config文件所有运行的实际profile
2. prod，在这个例子中可能用来生成完整的不包含软连接、使用严格编译选项的发布包
3. native，强制使用[HiPE](http://www.erlang.org/doc/man/HiPE_app.html) 编译，以获得更快的运算（数学运算处理）代码。
4. test，加载模拟库，测试运行中启用在文件中保存debug信息选项

可以通过多种方式组合，以下时一些例子：

1. `rebar3 ct`：运行项目的通用测试套件。按顺序，profiles会先应用default、然后是test，因为ct要求使用test这个profile。
2. `rebar3 as test ct`：和上面一样，profile只会应用一次。
3. `rebar3 as native ct`在native模式下运行测试。profiles应用顺序是default、然后是native、最后是test（最后由运行命令指定的）
4. `rebar3 as test,native ct`类似组合3，略有变化。应用profiles的时候，rebar3会将它们全部展开，按照正确的顺序应用。所以这里的顺序是：default、然后是test、然后是native。最后的test（由ct命令指定）因为已经应用过了所以被忽略。这里不完全等同于`rebar3 as native ct`，如果test和native有着互相冲突的配置项，那么profile的顺序则十分重要（下面《选项合并算法》有详述）。
5. `rebar3 release`：只使用default构建发行包
6. `rebar3 as prod release`：构建一个不包含开发模式、包含严格编译选项的发布包
7. `rebar3 as prod, native release`：构建一个和上一个一样、另外将模块编译成native模式的发布包
8. `rebar3 as prod release`，环境变量REBAR_PROFILE=native：构建一个如7的发布包，不过native的顺序在prod之前。

应用程序的profiles应用顺序为：

1. default
2. REBAR_PROFILE的值（如果有）
3. 作为命令行一部分的as指定的profile（从左到右）
4. 由每个命令指定的profile

因此profiles是一个在特殊上下文环境中指定配置子集的可组合的方式。

> ### 📘锁定依赖项
>
> 
>
> 只有顶层`rebar.config`中default这个profile指定的依赖项会保存到 `rebar.lock`文件，其它依赖不会获得锁定。
>
> 如果某人想 “为生产环境锁定” (指生产相关的 profiles)，答案是依赖项保持在default profile 并使用 [releases](https://www.rebar3.org/docs/releases), 它允许生产任意时刻能够复用的编译好的物件。

## 选项合并算法

尝试自动合并所有配置选项通常比较棘手。不同的工具或命令期望不同，或者是作为一个元组列表、属性列表、或者键/值对转化进一个按照某种顺序的字典。

为了尽可能支持最普遍的形式，rebar3把它们当作松散的属性列表和元组列表的组合，这意味着下面的选项全都视为拥有native键：

* native
* {native,{hipe,o3}}
* {native,still,supported}

以上形式的某些部分，工具可能支持也可能不支持。比如，Erlang编译器支持这样定义宏：`{d, 'MACRONAME'}`或者`{d, 'MACRONAME', MacroValue}`，但是单独的`d`不支持。而`native`和`{native,{hipe,o3}}`确实支持。

Rebar3正确支持所有这些形式并以功能性方式合并它们，让我们以下面的profiles为例：

```erlang
{profiles, [
    {prod, [
        {erl_opts, [no_debug_info, warnings_as_errors]},
    ]},
    {native, [
        {erl_opts, [{native, {hipe, o3}}, {d, 'NATIVE'}]}
    ]},
    {test, [
        {erl_opts, [debug_info]}
    ]}
]}.
```

按不同的顺序应用以上profiles会产生不同的`erl_opts`选项。

* `rebar3 as prod,native,test <command>`: [debug_info, {d, 'NATIVE'}, {native, {hipe, o3}}, no_debug_info, warnings_as_errors]
* `rebar3 as test,prod,native <command>`:[{d, 'NATIVE'}, {native, {hipe, o3}}, no_debug_info, warnings_as_errors, debug_info]
* `rebar3 as native,test,prod <command>`:[no_debug_info, warnings_as_errors, debug_info, {d, 'NATIVE'}, {native, {hipe, o3}}]
* `rebar3 as native,prod,test <command>`:[debug_info, no_debug_info, warnings_as_errors, {d, 'NATIVE'}, {native, {hipe, o3}}]

注意最后的profiles应用在列表的第一个元素，每个profiles的列表中的元素会按照它们的键排序。

这将允许rebar3命令以正确的顺序拾取元素，同时仍支持需要多个元素共享同一键的多值列表（例如[{d，'ABC'}，{d，'DEF'}] ，这是两个独立的宏！）。不支持重复元素的命令可以在<font color="red">第一个命令之后停止处理</font>，而那些从中构建字典（或Maps）的命令可以选择按原样插入，或者可以先安全地反转列表（如果最后处理的元素变为Maps中的最后一个）。

以这种方式安全地处理所有配置文件合并规则。插件作者应了解这些规则并制定相应的计划。

请注意，实际上，Erlang编译器不能与debug_info和no_debug_info配合使用（这不是真正的选项，而是由Rebar3添加的）。Rebar3做了一些魔术来删除这些特定值，以适应编译器，但是并没有将此温和扩展到所有工具，设计插件以使用{OptionName，true | false}通常是一个好主意。

>### 🚧依赖项和Profiles
>
>
>
>依赖项将始终在将prod这个profile应用于其配置的情况下进行编译。没有其它 (当然`default`例外) profile应用在任何依赖项上。.即使已将它们配置为prod，依赖仍将被获取到在其下声明的配置文件的profile目录中，比如说，顶层的 `deps`上的一个依赖会在 `_build/default/lib`目录而在 `test`这个profile下的依赖会被拉取到 `_build/test/lib`，而且都会使用 依赖项自己的`prod`这个 profile 编译。

