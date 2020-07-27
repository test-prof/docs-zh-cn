# 使用 StackProf 进行分析

[StackProf](https://github.com/tmm1/stackprof) 是用于 Ruby 的采样调用堆栈分析器。

## 教学

安装 `stackprof` gem (>= 0.2.9):

```ruby
# Gemfile
group :development, :test do
  gem "stackprof", ">= 0.2.9", require: false
end
```

StackProf 分析器有两种模式：`global` 和 `per-example`。

你可以使用环境变量 `TEST_STACK_PROF` 来激活全局分析：

```sh
TEST_STACK_PROF=1 bundle exec rake test

# or for RSpec
TEST_STACK_PROF=1 rspec ...
```

或在你的代码中这样写：

```ruby
TestProf::StackProf.run
```

TestProf 为 RSpec 提供了一个内置的 shared context 以对测试用例进行单独分析：

```ruby
it "is doing heavy stuff", :sprof do
  # ...
end
```

**注意：**在 global 被激活时，per-example 分析是无法工作的。

## 报告格式

Stackprof 提供了一个 CLI 工具对生成的报告进行操作（比如，转换为不同格式）。

默认情况下，TestProf 为你展示的命令\* 以生成分析火焰图的 HTML 报告，这样你可以自己打开它。

\* 仅当你收集了_原生_ 用例数据时，而这是 TestProf 的默认行为。

有时侯 JSON 报告是很有用的（比如，跟 [speedscope](https://www.speedscope.app)一起使用时），但 `stackprof` 仅自 [0.2.13](https://github.com/tmm1/stackprof/blob/master/CHANGELOG.md#0213) 版本起才支持。

如果你正在使用旧的 Stackprof 版本，TestProf 可帮助从_原生_数据生成 JSON 报告。对此，可使用 `TEST_STACK_PROF_FORMAT=json` 或在你代码中配置默认格式：

```ruby
TestProf::StackProf.configure do |config|
  config.format = "json"
end
```

## 分析启动时间

应用的启动时间也会让测试变慢。尝试以如下命令使用 StackProf 来分析启动过程：

```sh
TEST_STACK_PROF=boot rspec ./spec/some_spec.rb
```

**注意：** 我们推荐使用火焰图来分析启动时间，这正是为什么原生数据收集总是在`boot`模式中被开启的原因。

## 配置

你可以通过 `TEST_STACK_PROF_MODE` 环境变量更改 StackProf 模式（默认是`wall`）。

你也可以通过 `TEST_STACK_PROF_INTERVAL` 环境变量更改 StackProf 的间隔时间。对于 `wall` 和 `cpu` 模式，`TEST_STACK_PROF_INTERVAL` 表示微秒，并且默认每个 `stackprof` 为 1000。
对于`object`模式，`TEST_STACK_PROF_INTERVAL` 表示分配，并且默认每个 `stackprof` 为 1。

参看 [stack_prof.rb](https://github.com/test-prof/test-prof/tree/master/lib/test_prof/stack_prof.rb) 了解其所有可用的配置选项及其用法。
