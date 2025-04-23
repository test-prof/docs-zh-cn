# 与 Ruby 分析器一起使用

Test Prof 允许你使用通用的 Ruby 分析器来分析测试套件，而无需自己编写任何分析代码。只需安装 profiler 库并运行测试即可！

支持的 profilers:

- [StackProf](https://github.com/tmm1/stackprof)
- [Vernier](https://www.speedscope.app)
- [RubyProf](https://github.com/jhawthorn/vernier)

## StackProf

[StackProf][https://github.com/tmm1/stackprof] 是 Ruby 的采样调用堆栈分析器。

确保你的依赖项中有 `stackprof` ：

```ruby
# Gemfile
group :development, :test do
  gem "stackprof", ">= 0.2.9", require: false
end
```

### 使用 StackProf 分析整个测试套件

注意：建议使用[测试采样](https://github.com/test-prof/test-prof/blob/master/docs/recipes/tests_sampling.md)来生成较小的分析报告。

你可以通过设置 env 变量 `TEST_STACK_PROF` 来激活 StackProf 分析：

```sh
TEST_STACK_PROF=1 bundle exec rake test

# or for RSpec
TEST_STACK_PROF=1 bundle exec rspec ...
```

在测试运行结束时，你将看到来自 Test Prof 的消息，其中包括生成报告的路径（原始 StackProf 格式和 JSON）：

```sh
...

[TEST PROF INFO] StackProf report generated: tmp/test_prof/stack-prof-report-wall-raw-total.dump
[TEST PROF INFO] StackProf JSON report generated: tmp/test_prof/stack-prof-report-wall-raw-total.json
```

我们建议将 JSON 报告上传到 [Speedscope](https://www.speedscope.app/) 并分析火焰图。否则，请随意使用 `stackprof` CLI 来操作原始报告。

### 使用 StackProf 分析单个示例

Test Prof 为 RSpec 提供了一个内置的 shared context，用于单独分析示例：

```ruby
it "is doing heavy stuff", :sprof do
  # ...
end
```

注意：当全局（每个套件）分析被激活时，每个示例的分析不起作用。

### 使用 StackProf 分析应用程序启动

应用程序启动时间也可能使测试速度变慢。尝试使用以下命令通过 StackProf 分析你的启动过程：

```sh
# pick some random spec (1 is enough)
$ TEST_STACK_PROF=boot bundle exec rspec ./spec/some_spec.rb

...
[TEST PROF INFO] StackProf report generated: tmp/test_prof/stack-prof-report-wall-raw-boot.dump
[TEST PROF INFO] StackProf JSON report generated: tmp/test_prof/stack-prof-report-wall-raw-boot.json
```

### StackProf 配置

你可以通过 `TEST_STACK_PROF_MODE` 环境变量来更改 StackProf 模式（默认是`wall`）。

你还可以通过 `TEST_STACK_PROF_INTERVAL` 环境变量更改 StackProf 间隔时间。对于 `wall` 和 `cpu` 模式，`TEST_STACK_PROF_INTERVAL` 表示微秒，默认每个 `stackprof` 为 1000 。对于 `object` 模式，`TEST_STACK_PROF_INTERVAL` 表示分配，默认每个 `stackprof` 为 1。

你可以通过设置 `TEST_STACK_PROF_IGNORE_GC` 环境变量来禁用垃圾回收。垃圾回收时间仍将存在于配置文件中，但不会使用它自己的帧来显式标记。

请参阅 [stack_prof.rb](https://github.com/test-prof/test-prof/tree/master/lib/test_prof/stack_prof.rb) 了解所有可用的配置选项及其用法。

## Vernier

[Vernier][https://github.com/jhawthorn/vernier] 是 Ruby 的下一代采样分析器。试一试，看看它是否有助于识别测试性能瓶颈！

确保你的依赖项中有 `vernier` ：

```ruby
# Gemfile
group :development, :test do
  gem "vernier", ">= 0.3.0", require: false
end
```

### 使用 Vernier 分析整个测试套件

注意：建议使用[测试采样](https://github.com/test-prof/test-prof/blob/master/docs/recipes/tests_sampling.md)来生成较小的分析报告。

你可以通过设置环境变量 `TEST_VERNIER` 来激活 Verner 分析：

```sh
TEST_VERNIER=1 bundle exec rake test

# or for RSpec
TEST_VERNIER=1 bundle exec rspec ...
```

在测试运行结束时，你将看到来自 Test Prof 的消息，其中包括生成报告的路径：

```sh
...

[TEST PROF INFO] Vernier report generated: tmp/test_prof/vernier-report-wall-raw-total.json
```

使用 [profile-viewer](https://github.com/tenderlove/profiler/tree/ruby) gem 或将你的配置文件上传到 [vernier.prof](https://vernier.prof/)。或者，你也可以使用 [profiler.firefox.com](https://profiler.firefox.com/)，profile-viewer 是它的 fork。

### 使用 Vernier 分析单个示例

Test Prof 为 RSpec 提供了一个内置的 shared context，用于单独分析示例：

```ruby
it "is doing heavy stuff", :vernier do
  # ...
end
```

注意：当全局（每个套件）分析被激活时，每个示例的分析不起作用。

### 使用 Vernier 分析应用程序启动

你也可以分析应用程序启动过程：

```sh
# pick some random spec (1 is enough)
TEST_VERNIER=boot bundle exec rspec ./spec/some_spec.rb
```

### 从 Active Support Notifications 添加标记

可以通过从 Active Support Notifications 添加事件标记来向生成的报告添加更多见解：

```
TEST_VERNIER=1 TEST_VERNIER_HOOKS=rails bundle exec rake test

# or for RSpec
TEST_VERNIER=1 TEST_VERNIER_HOOKS=rails bundle exec rspec ...
```

或者你可以通过 `Vernier` 配置设置 hooks 参数：

```
TestProf::Vernier.configure do |config|
  config.hooks = :rails
end
```

## RubyProf

轻松将 [ruby-prof](https://github.com/ruby-prof/ruby-prof) 的强大功能集成到测试套件中。

确保 `ruby-prof` 已安装：

```ruby
# Gemfile
group :development, :test do
  gem "ruby-prof", ">= 1.4.0", require: false
end
```

### 使用 RubyProf 分析整个测试套件

注意：强烈建议使用[测试采样](https://github.com/test-prof/test-prof/blob/master/docs/recipes/tests_sampling.md)来生成较小的分析报告，并避免缓慢的测试运行（RubyProf 有很大的开销）。

你可以使用环境变量 `TEST_RUBY_PROF` 激活全局分析：

```sh
TEST_RUBY_PROF=1 bundle exec rake test

# or for RSpec
TEST_RUBY_PROF=1 bundle exec rspec ...
```

在测试运行结束时，您将看到来自 Test Prof 的消息，其中包括生成报告的路径：

```sh
[TEST PROF INFO] RubyProf report generated: tmp/test_prof/ruby-prof-report-flat-wall-total.txt
```

#### 跳过测试套件引导

**注意：** 仅限 RSpec。

从 RubyProf 报告中排除应用程序启动和测试负载以仅分析正在执行的测试可能很有用（这样就不会让 `Kernel#require` 成为最慢的方法之一）。

为此，请指定 `TEST_RUBY_PROF_BOOT=false`（或 “0”，或 “f”）env 变量：

```
$ TEST_RUBY_PROF=1 TEST_RUBY_PROF_BOOT=0 bundle exec rspec ...

[TEST PROF] RubyProf enabled for examples

...
```

### 使用 RubyProf 分析单个示例

TestProf 为 RSpec 提供了一个内置的 shared context，用于单独分析示例：

```ruby
it "is doing heavy stuff", :rprof do
  # ...
end
```

注意：当全局（每个套件）分析被激活时，每个示例的分析不起作用。

### RubyProf 配置

最有用的配置选项是 `printer` – 它允许你指定 RubyProf [printer](https://github.com/ruby-prof/ruby-prof#printers)。

你可以通过环境变量 `TEST_RUBY_PROF` 指定 printer

```sh
TEST_RUBY_PROF=call_stack bundle exec rake test
```

或者在代码中：

```ruby
TestProf::RubyProf.configure do |config|
  config.printer = :call_stack
end
```

默认情况下，我们使用 `FlatPrinter` .

注意：要为每个示例配置文件指定 printer，请使用 `TEST_RUBY_PROF_PRINTER` 环境变量（因为使用 `TEST_RUBY_PROF` 激活了全局分析）。

此外，你还可以通过 `TEST_RUBY_PROF_MODE` 环境变量指定 RubyProf 模式（`wall` 、 `cpu` 等）。

请参阅 [ruby_prof.rb](https://github.com/test-prof/test-prof/tree/master/lib/test_prof/ruby_prof.rb) 了解所有可用的配置选项及其用法。

从配置文件中排除某些方法以仅关注应用程序代码非常有用。

TestProf 默认使用 RubyProf `exclude_common_methods!` （通过 `config.exclude_common_methods = false` 来禁用 ）。

默认情况下，我们排除了一些其他常用方法和 RSpec 特定的内部方法。要禁用 TestProf 定义的排除项，请设置 `config.test_prof_exclusions_enabled = false` 。

你可以通过 `config.custom_exclusions` 指定自定义排除项，例如：

```ruby
TestProf::RubyProf.configure do |config|
  config.custom_exclusions = {User => %i[save save!]}
end
```
