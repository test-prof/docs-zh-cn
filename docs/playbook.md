# 快速手册

本文档旨在帮助你开始分析测试套件，并回答以下问题：首先运行哪些配置文件？我们如何解释结果以选择后续步骤？等等。

注意：本文档假设你使用的是 Ruby on Rails 和 RSpec 测试框架。这些思想可以很容易地转化为其他框架。

## 步骤 0：基础配置

一些显而易见的基础：

- 在测试中禁用日志记录 —— 它是无用的。如果你确实需要它，请使用我们的[日志记录工具](https://github.com/test-prof/test-prof/blob/master/docs/recipes/logging.md)。

```ruby
config.logger = ActiveSupport::TaggedLogging.new(Logger.new(nil))
config.log_level = :fatal
```

- 默认情况下禁用覆盖率和内置分析。使用 env var 来启用它（例如， `COVERAGE=true` ）

## 步骤 1：通用分析

它有助于识别不那么容易实现的结果。我们建议使用 [StackProf](https://github.com/test-prof/test-prof/blob/master/docs/profilers/stack_prof.md)，因此你必须先安装它（如果没有的话）：

```sh
bundle add stackprof
```

默认情况下，将 Test Prof 配置为生成 JSON 配置文件：

```ruby
TestProf::StackProf.configure do |config|
  config.format = "json"
end
```

我们建议使用 [speedscope](https://www.speedscope.app/) 来分析这些配置文件。

### 步骤 1.1：应用程序启动分析

```sh
TEST_STACK_PROF=boot rspec ./spec/some_spec.rb
```

注意：运行单个 spec/test 就足以进行此分析。

看到什么了？下面是些例子：

- 未使用或未配置 [Bootsnap](https://github.com/Shopify/bootsnap) 来缓存所有内容（例如 YAML 文件）
- 测试中不需要的拖慢 Rails 的初始化项。

### 步骤 1.2.抽样测试分析

其思想是多次运行测试的随机子集，以揭示一些应用程序范围内的问题。你必须先启用[采样功能](https://github.com/test-prof/test-prof/blob/master/docs/recipes/tests_sampling.md)：

```rb
# For RSpec in your spec_helper.rb
require "test_prof/recipes/rspec/sample"

# For Minitest in your test_helper.rb
require "test_prof/recipes/minitest/sample"
```

然后多次运行并分析获得的火焰图：

```sh
SAMPLE=100 bin/rails test
# or
SAMPLE=100 bin/rspec
```

通常会发现到的：

- 加密调用（ `*crypt*` -任何东西）：放宽其在测试环境中的设置
- 日志调用：你确定禁用了日志吗？
- 数据库：也许有一些现成的经验（比如对每个测试都使用 DatabaseCleaner truncation 而非 transaction）
- 网络请求：不应该用于单元测试，对于浏览器测试来说是不可避免的；使用 [Webmock](https://github.com/bblimke/webmock) 来完全禁用 HTTP 请求。

## 步骤 2：缩小范围

对于大型代码库来说，这是一个重要的步骤。我们必须优先考虑能带来最大价值（减少时间）的快速修复，而不是单独处理复杂、缓慢的测试（即使它们是最慢的测试）。为此，我们要首先确定对整体运行时间贡献最大的测试类型。

为此我们使用 [TagProf](https://github.com/test-prof/test-prof/blob/master/docs/profilers/tag_prof.md)：

```sh
TAG_PROF=type TAG_PROF_FORMAT=html TAG_PROF_EVENT=sql.active_record,factory.create bin/rspec
```

查看生成的图表，你可以确定两种最耗时的测试类型（通常是 model 和/或 controller）。

We assume that it's easier to find a common slowness cause for the whole group and fix it than dealing with individual tests. Given that assumption, we continue the process only within the selected group (let's say, models).

我们假设为整个团队找到一个常见的缓慢原因并修复它，这比处理单个测试更容易。鉴于该假设，我们仅在选定的组（例如 model）中继续该过程。

## 步骤 3：专门的分析

在选定的组中，我们可以首先通过 [EventProf](https://github.com/test-prof/test-prof/blob/master/docs/profilers/event_prof.md) 执行基于事件的快速分析。（也许也启用了采样）。

### 步骤 3.1：依赖项配置

此时，我们可能会发现一些配置错误或误用的依赖项或 Gem。常见示例：

- Inlined Sidekiq jobs:

```sh
EVENT_PROF=sidekiq.inline bin/rspec spec/models
```

- Wisper broadcasts ([需要补丁](https://gist.github.com/palkan/aa7035cebaeca7ed76e433981f90c07b)):

```sh
EVENT_PROF=wisper.publisher.broadcast bin/rspec spec/models
```

- PaperTrail 日志创建：

启用自定义分析：

```rb
TestProf::EventProf.monitor(PaperTrail::RecordTrail, "paper_trail.record", :record_create)
TestProf::EventProf.monitor(PaperTrail::RecordTrail, "paper_trail.record", :record_destroy)
TestProf::EventProf.monitor(PaperTrail::RecordTrail, "paper_trail.record", :record_update)
```

运行测试：

```sh
EVENT_PROF=paper_trail.record bin/rspec spec/models
```

请参阅 [Sidekiq 示例](https://evilmartians.com/chronicles/testprof-a-good-doctor-for-slow-ruby-tests#background-jobs)，了解如何使用 [RSpecStamp](https://github.com/test-prof/test-prof/blob/master/docs/recipes/rspec_stamp.md) 快速修复此类问题。

### 步骤 3.2：数据生成

I根据在数据库或 factories（如果有的话）中花费的时间来识别出最慢的测试：

```sh
# Database interactions
EVENT_PROF=sql.active_record bin/rspec spec/models

# Factories
EVENT_PROF=factory.create bin/rspec spec/models
```

现在，我们可以将范围进一步缩小到生成的报告中的前 10 个文件。如果你使用 factories，请使用 `factory.create` 报表。

提示：在 RSpec 中，你可以运行以下命令自动使用自定义 tag 来标记最慢的示例：

```sh
EVENT_PROF=factory.create EVENT_PROF_STAMP=slow:factory bin/rspec spec/models
```

## 步骤 4：Factories 的使用

在测试中通过 `slow:factory` 找出最常用的 factories：

```sh
FPROF=1 bin/rspec --tag slow:factory
```

如果你看到某些 factories 的使用次数远远超过示例总数，则处理_factory cascades_。

可视化 cascades:

```sh
FPROF=flamegraph bin/rspec --tag slow:factory
```

可视化应该有助于确定要修复的 factories。你可以在[这篇文章](https://evilmartians.com/chronicles/testprof-2-factory-therapy-for-your-ruby-tests-rspec-minitest)中找到可能的解决方案。

### 步骤 4.1：Factory 默认设置

修复因模型关联而生成的 cascades 的一个选项是使用 [Factory 默认设置](https://github.com/test-prof/test-prof/blob/master/docs/recipes/factory_default.md)。若要估计潜在影响并确定要应用此模式的 factories，请运行以下分析器：

```sh
FACTORY_DEFAULT_PROF=1 bin/rspec --tag slow:factory
```

尝试添加 `create_default` 来衡量影响：

```sh
FACTORY_DEFAULT_SUMMARY=1 bin/rspec --tag slow:factory

# More hits — better
FactoryDefault summary: hit=11 miss=3
```

### 步骤 4.2：Factory fixtures

回到 `FPROF=1` 结果报表，查看是否为每个示例创建了一些记录（通常为 `user` 、 `account` `team` ）。考虑使用 [AnyFixture](https://github.com/test-prof/test-prof/blob/master/docs/recipes/any_fixture.md) 将它们替换为 fixtures。

## 步骤 5：可重用的设置

在多个用例中共享相同的设置很常见。可以使用 [RSpecDissect](https://github.com/test-prof/test-prof/blob/master/docs/profilers/rspec_dissect.md) 衡量 `let` 或 `before` 所花费的时间与实际时间的比较：

```sh
RD_PROF=1 bin/rspec
```

看看最慢的组，并尝试用 [let_it_be](./recipes/let_it_be.md) 替换 `let/let!`，和用 [before_all](./recipes/before_all.md) 替换 `before`。

**重要提示**：Knapsack Pro 用户必须注意，每个示例的平衡消除了使用 `let_it_be` / `before_all` 的积极影响。你必须切换到每个文件的平衡，同时保持文件较小 —— 这样才能最大限度地发挥 Test Prof 优化的效果。

## 结论

将上述步骤应用于给定的一组测试后，你应该开发针对代码库优化的模式和技术。然后，你只需将它们推广到其他组即可。祝你好运！
