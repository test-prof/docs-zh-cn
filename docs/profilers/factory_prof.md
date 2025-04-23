# Factory 分析器

FactoryProf 追踪你的 factories 的使用统计数据，比如，每个 factory 的使用频率。

输出范例：

```sh
[TEST PROF INFO] Factories usage

Total: 15285
Total top-level: 10286
Total time: 299.5937s
Total uniq factories: 119

 total   top-level   total time    time per call      top-level time            name
  6091        2715    115.7671s          0.0426s            50.2517s            user
  2142        2098     93.3152s          0.0444s            92.1915s            post
  ...
```

它展示了 factory 运行的总数和 _top-level_ 运行的数量，比如，不在另一个 factory 调用期间（如在使用关联关系时）。

它还显示使用 Factory 生成记录所花费的时间以及每次 Factory 调用所花费的时间。

**注意**：FactoryProf 仅追踪数据库持久化的 factories。对于 FactoryGirl / FactoryBot，是通过使用 `create` 策略所提供的 factories。对于 Fabrication，是使用 `create` 方法所创建的对象。

## 教学

FactoryProf 可跟 FactoryGirl / FactoryBot 或者 Fabrication 一起使用——应用程序可以同时安装这两个 gem 使用。

使用 `FPROF` 环境变量来激活 FactoryProf：

```sh
# Simple profiler
FPROF=1 rspec

# or
FPROF=1 bundle exec rake test
```

### [*Nate Heckler*](https://twitter.com/nateberkopec/status/1389945187766456333) 模式

为鼓励你尽快修复 factories，我们还提供了一个特殊的 *Nate heckler* 模式。

将如下添加到 `rails_helper.rb` 或 `test_helper.rb` 中：

```
require "test_prof/factory_prof/nate_heckler"
```

对于每个测试运行，请参阅 factories 的整体使用情况：

```
[TEST PROF INFO] Time spent in factories: 04:31.222 (54% of total time)
```

###  变体

还可以通过提供 `FPROF_VARS=1` 环境变量或在代码中启用它，向报告添加*变体* （如 traits、overrides）信息：

```
TestProf::FactoryProf.configure do |config|
  config.include_variations = true
end
```

例如：

```
$ FPROF=1 FPROF_VARS=1 bin/rails test

...

[TEST PROF INFO] Factories usage

Total: 15285
Total top-level: 10286
Total time: 04:31.222 (out of 07.16.124)
Total uniq factories: 119

   name           total   top-level   total time    time per call      top-level time

   user            6091        2715    115.7671s          0.0426s             50.251s
     -             5243        1989      84.231s          0.0412s             34.321s
     .admin         823         715      15.767s          0.0466s              5.257s
     [name,role]     25          11       7.671s          0.0666s              1.257s
   post            2142        2098      93.315s          0.0444s             92.191s
     _             2130        2086      87.685s          0.0412s             88.191s
     .draft[tags]    12          12       9.315s           0.164s             42.115s
   ...
```

上面示例中, `-` 表示 factory 没有 traits 或 overrides (例如 `create(:user)`), `.xxx` 表示 trait，而 `[a,b]` 表示 overrides keys，例如 `create(:user, :admin)` 是一个 `.admin` 变体，而 `create(:post, :draft, tags: ["a"])`—`.draft[tags]`

#### 变体限制配置

运行 FactoryProf 时，输出可能包含太长的变体，这将使输出失真。

为避免这种情况并专注于最重要的统计数据，可以指定变体限制的值。然后，将显示一个特殊 ID （`[...]`），而不是 traits / overrides 数量超过限制的变体。

要使用变体限制参数，将环境变量 `FPROF_VARIATIONS_LIMIT` 设置为 `N`（`N` 是限制数）：

```
FPROF=1 FPROF_VARIATIONS_LIMIT=5 rspec

# or
FPROF=1 FPROF_VARIATIONS_LIMIT=5 bundle exec rake test
```

或者可以通过 `FactoryProf` 配置设置 limit 参数：

```
TestProf::FactoryProf.configure do |config|
  config.variations_limit = 5
end
```

### 将分析结果导出为 JSON 文件

FactoryProf 可以将配置文件结果保存为 JSON 文件。

要使用此功能，请将 `FPROF` 环境变量设置为 `json`：

```
FPROF=json rspec

# or
FPROF=json bundle exec rake test
```

输出示例：

```
[TEST PROF INFO] Profile results to JSON: tmp/test_prof/test-prof.result.json
```

### 降低输出

运行 FactoryProf 时，输出可能包含大量已使用过多次的 factories 的行。为避免这种情况并专注于最重要的统计数据，可以指定阈值。然后，你将看到总数超过阈值的 factories。

要使用 threshold 选项，`FPROF_THRESHOLD` 环境变量设置为 `N`（`N` 是阈值数字）：

```
FPROF=1 FPROF_THRESHOLD=30 rspec

# or
FPROF=1 FPROF_THRESHOLD=30 bundle exec rake test
```

或者，你可以通过 `FactoryProf` 配置设置 threshold 参数：

```
TestProf::FactoryProf.configure do |config|
  config.threshold = 30
end
```

### 截断 factory 长名称

When running FactoryProf on a codebase with long factory names, the table layout may break. To avoid this you can allow FactoryProf to truncate these names.
在具有较长 factory 名称的代码库上运行 FactoryProf 时，报表布局可能会打乱。为避免这种情况，可以允许 FactoryProf 截断这些名称。

要使用截断，请将环境变量 `FPROF_TRUNCATE_NAMES` 设置为 `1`：

```
FPROF=1 FPROF_TRUNCATE_NAMES=1 rspec

# or
FPROF=1 FPROF_TRUNCATE_NAMES=1 bundle exec rake test
```

或者你可以通过 `FactoryProf` 配置设置 truncate_names 参数：

```
TestProf::FactoryProf.configure do |config|
  config.truncate_names = true
end
```

## Factory 火焰图

FactoryProf 最有用的特性就是 _FactoryFlame_ 报告。这是对 Brendan Gregg's 的 [火焰图](http://www.brendangregg.com/flamegraphs.html) 的特殊解读，让你可以识别出 _factory cascades_.

要生成 FactoryFlame 报告，把 `FPROF` 环境变量设置为 `flamegraph`：

```sh
FPROF=flamegraph rspec

# or
FPROF=flamegraph bundle exec rake test
```

报告看起来这样：

<img alt="FactoryProf: Flamegraph report" data-origin="/assets/factory-flame.gif" src="/assets/factory-flame.gif">

如何解读它？

每一栏指示一个 _factory stack_ 或 _cascade_，即一个递归 `#create` 方法调用的序列。考虑下面这个范例：

```ruby
factory :comment do
  answer
  author
end

factory :answer do
  question
  author
end

factory :question do
  author
end

create(:comment) #=> creates 5 records

# And the corresponding stack is:
# [:comment, :answer, :question, :author, :author, :author]
```

栏越宽，其堆栈出现的频率越高。

`root` 单元格展示了 `create` 调用的总数。

## 感谢

- [Martin Spier](https://github.com/spiermar) 的 [d3-flame-graph](https://github.com/spiermar/d3-flame-graph)

- [Sam Saffron](https://github.com/SamSaffron) 的 [flame graphs implementation](https://github.com/SamSaffron/flamegraph).
