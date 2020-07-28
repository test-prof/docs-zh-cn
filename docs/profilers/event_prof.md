# EventProf

EventProf 在你的测试套件运行期间收集各种测量指标（比如 ActiveSupport::Notifications）。

它的工作很类似于 `rspec --profile` 但可追踪任意事件。

输出范例：

```sh
[TEST PROF INFO] EventProf results for sql.active_record

Total time: 00:00.256 of 00:00.512 (50.00%)
Total events: 1031

Top 5 slowest suites (by time):

AnswersController (./spec/controllers/answers_controller_spec.rb:3) – 00:00.119 (549 / 20) of 00:00.200 (59.50%)
QuestionsController (./spec/controllers/questions_controller_spec.rb:3) – 00:00.105 (360 / 18) of 00:00.125 (84.00%)
CommentsController (./spec/controllers/comments_controller_spec.rb:3) – 00:00.032 (122 / 4) of 00:00.064 (50.00%)

Top 5 slowest tests (by time):

destroys question (./spec/controllers/questions_controller_spec.rb:38) – 00:00.022 (29) of 00:00.064 (34.38%)
change comments count (./spec/controllers/comments_controller_spec.rb:7) – 00:00.011 (34) of 00:00.022 (50.00%)
change Votes count (./spec/shared_examples/controllers/voted_examples.rb:23) – 00:00.008 (25) of 00:00.022 (36.36%)
change Votes count (./spec/shared_examples/controllers/voted_examples.rb:23) – 00:00.008 (32) of 00:00.035 (22.86%)
fails (./spec/shared_examples/controllers/invalid_examples.rb:3) – 00:00.007 (34) of 00:00.014 (50.00%)

```

## 教学

目前，EventProf 仅支持 ActiveSupport::Notifications。

激活 EventProf：

### RSpec

使用 `EVENT_PROF` 环境变量设置为事件名称：

```sh
# Collect SQL queries stats for every suite and example
EVENT_PROF='sql.active_record' rspec ...
```

你可以同时追踪多个事件：

```sh
EVENT_PROF='sql.active_record,perform.active_job' rspec ...
```

### Minitest

使用 `EVENT_PROF` 环境变量设置为事件名称：

```sh
# Collect SQL queries stats for every suite and example
EVENT_PROF='sql.active_record' rake test
```

或者也可以使用 CLI 选项：

```sh
# Run a specific file using CLI option
ruby test/my_super_test.rb --event-prof=sql.active_record

# Show the list of possible options:
ruby test/my_super_test.rb --help
```

### 与 Minitest::Reporters 一起使用

如果在你的项目中使用了 `Minitest::Reporters`，那么你必须明确在测试帮助方法文件中声明它：

```sh
require 'minitest/reporters'
Minitest::Reporters.use! [YOUR_FAVORITE_REPORTERS]
```

#### 注意

当你以 gem 安装了 `minitest-reporters` 但并未在你的 `Gemfile` 里声明时，
确保总是在测试运行命令之前使用 `bundle exec` （但我们相信你总会这样做的）。
否则，你会得到由 Minitest plugin 系统造成的报错, 该系统对所有的 `minitest/*_plugin.rb` 扫描
`$LOAD_PATH` 内的入口, 因此该场景中可用的 `minitest-reporters` plugin 的初始化就不正确了。

如果你在用 Rails，参看 [Rails guides](http://guides.rubyonrails.org/active_support_instrumentation.html)来了解所有可用的事件。

如果你在用 [rom-rb](http://rom-rb.org)，那么可能会对分析`'sql.rom'` 事件感兴趣。

## 配置

默认情况下，EventProf 仅收集关于 top-level 组的信息（aka suites），
但你也可以分析单独的用例。只用设置好配置选项：

```ruby
TestProf::EventProf.configure do |config|
  config.per_example = true
end
```

或者提供 `EVENT_PROF_EXAMPLES=1` 环境变量。

另一个有用但配置参数是——`rank_by`。它负责对统计数字排序——或者是事件耗费时间或者是出现次数：

```sh
EVENT_PROF_RANK=count EVENT_PROF='instantiation.active_record' be rspec
```

参看 [event_prof.rb](https://github.com/test-prof/test-prof/tree/master/lib/test_prof/event_prof.rb) 了解所有可用配置选项及其用法。

## 与 RSpecStamp 一起使用

EventProf 可跟 [RSpec Stamp](../recipes/rspec_stamp.md) 一起使用来以自定义 tag 自动标识_慢_测试用例，比如：

```sh
EVENT_PROF="sql.active_record" EVENT_PROF_STAMP="slow:sql" rspec ...
```

运行上面命令之后，最慢的测试用例组（及测试用例，如果配置的话）就会用 `slow: :sql` 的 tag 进行标识。

## 自定义测量器

要让 EventProf 和你的测量器引擎一起使用，只用完成下面两步：

- 为你的测量器添加一个封装器：

```ruby
# Wrapper over your instrumentation
module MyEventsWrapper
  # Should contain the only one method
  def self.subscribe(event)
    raise ArgumentError, "Block is required!" unless block_given?

    ::MyEvents.subscribe(event) do |start, finish, *|
      yield (finish - start)
    end
  end
end
```

- 在配置中设定测量器：

```ruby
TestProf::EventProf.configure do |config|
  config.instrumenter = MyEventsWrapper
end
```

## 自定义事件

### `"factory.create"`

FactoryGirl 提供了两个它自己的测量器（`factory_girl.run_factory`），但有一个警告——每次一个 factory 被使用时就会触发一个事件，即使我们是为嵌套关联关系使用 factory。因此由于重复计算这就不可能计算 factories 所耗费的时间了。

EventProf 为 FactoryGirl 带来了一个小的补丁，其仅为 top-level 的 `FactoryGirl.create` 调用提供测量器。如果你使用 `"factory.create"` 事件，它会自动加载：

```sh
EVENT_PROF=factory.create bundle exec rspec
```

> 自 v0.9.0 起

也支持 Fabrication（追踪隐式和显式的 `Fabricate.create` 调用）。

### `"sidekiq.jobs"`

收集关于 inline 运行的 Sidekiq jobs 的统计数字：

```sh
EVENT_PROF=sidekiq.jobs bundle exec rspec
```

**注意**： 会自动把 `rank_by` 设为 `count`（因为收集耗费时间的信息没有意义——参看下边）。

### `"sidekiq.inline"`

收集关于 inline 运行的 Sidekiq jobs 的统计数字（除去嵌套 jobs）：

```sh
EVENT_PROF=sidekiq.inline bundle exec rspec
```

使用该事件来分析正在运行的 Sidekiq jobs 的耗费时间。

## 分析任意方法

你也可以添加自定义事件来分析特定的方法（例如，在用 [RubyProf](./ruby_prof.md) 或 [StackProf](./stack_prof.md)找出一些最热的调用后）。

比如，有个 class 做了些很繁重的工作：

```ruby
class Work
  def do_smth(*args)
    # do something
  end
end
```

你可以通过添加一个 _monitor_ 来分析：

```ruby
# provide a class, event name and methods to monitor
TestProf::EventProf.monitor(Work, "my.work", :do_smth)
```

然后象平常一样运行 EventProf：

```sh
EVENT_PROF=my.work bundle exec rake test
```

> 自 v0.9.0 起

你也可以提供额外的选项：

- `top_level: true | false` （默认为`false`）：定义是否只考虑 top-level 调用并忽略此事件的嵌套触发器（这正是 "factory.create" 如何[实现的](https://github.com/test-prof/test-prof/blob/master/lib/test_prof/event_prof/custom_events/factory_create.rb)）。
- `guard: Proc` （默认为 `nil`）： 提供一个 Proc，防止触发一个事件：仅当 `guard` 返回 `true`时方法才被测量； `guard` 使用 `instance_exec`执行，并将方法参数传递给它。

例如：

```ruby
TestProf::EventProf.monitor(
  Sidekiq::Client,
  "sidekiq.inline",
  :raw_push,
  top_level: true,
  guard: ->(*) { Sidekiq::Testing.inline? }
)
```

你可以根据_需要_添加 monitors（比如仅当你想要追踪特定事件时），通过把代码封装到 `TestProf::EventProf::CustomEvents.register`方法内：

```ruby
TestProf::EventProf::CustomEvents.register("my.work") do
  TestProf::EventProf.monitor(Work, "my.work", :do_smth)
end

# Then call `activate_all` with the provided event
TestProf::EventProf::CustomEvents.activate_all(TestProf::EventProf.config.event)
```

这个 block 仅当特定事件在 EventProf 被启动时才执行。