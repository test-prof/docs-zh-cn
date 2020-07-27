# 使用 RubyProf 进行分析

易于把强大的 [ruby-prof](https://github.com/ruby-prof/ruby-prof) 整合进你的测试套件中。

## 教学

安装 `ruby-prof` gem (>= 0.17)：

```ruby
# Gemfile
group :development, :test do
  gem "ruby-prof", ">= 0.17.0", require: false
end
```

RubyProf 分析器有两种模式： `global` 和 `per-example`。

你可以使用环境变量`TEST_RUBY_PROF` 来激活全局分析：

```sh
TEST_RUBY_PROF=1 bundle exec rake test

# or for RSpec
TEST_RUBY_PROF=1 rspec ...
```

或在你的代码中这样写：

```ruby
TestProf::RubyProf.run
```

TestProf 为 RSpec 提供了一个内置的 shared context 以对测试用例进行单独分析：

```ruby
it "is doing heavy stuff", :rprof do
  # ...
end
```

**注意：**在 global 被激活时，per-example 分析是无法工作的。

## 配置

最有用的配置选项是 `printer` —— 它允许你指定 RubyProf 的 [printer](https://github.com/ruby-prof/ruby-prof#printers)。

你可以通过环境变量 `TEST_RUBY_PROF` 指定一个 printer：

```sh
TEST_RUBY_PROF=call_stack bundle exec rake test
```

或在你的代码中这样写：

```ruby
TestProf::RubyProf.configure do |config|
  config.printer = :call_stack
end
```

默认情况下，我们使用 `FlatPrinter`。

**注意：** 要为 per-example 分析指定 printer 请使用 `TEST_RUBY_PROF_PRINTER` 环境变量（因为使用 `TEST_RUBY_PROF` 会激活 global 分析）。

你也可以通过 `TEST_RUBY_PROF_MODE` 环境变量指定 RubyProf 的模式（`wall`, `cpu`, 等等）。

参看 [ruby_prof.rb](https://github.com/test-prof/test-prof/tree/master/lib/test_prof/ruby_prof.rb) 了解其所有可用的配置选项及其用法。

### 方法排除

从分析中排除某些方法以便仅专注于应用代码是很有用的。

TestProf 默认使用 RubyProf 的 [`exclude_common_methods!`](https://github.com/ruby-prof/ruby-prof/blob/e087b7d7ca11eecf1717d95a5c5fea1e36ea3136/lib/ruby-prof/profile/exclude_common_methods.rb) （使用 `config.exclude_common_methods = false`来禁用它）。

我们默认排除了一些其他通用方法和 RSpec 特别的内部方法。

要禁用 TestProf 定义的排除，请设置 `config.test_prof_exclusions_enabled = false`。

你可以通过 `config.custom_exclusions` 来指定自定义的排除，比如：

```ruby
TestProf::RubyProf.configure do |config|
  config.custom_exclusions = {User => %i[save save!]}
end
```
