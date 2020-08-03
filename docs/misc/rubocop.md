# 自定义 RuboCop Cops

TestProf 自带 [RuboCop](https://github.com/bbatsov/rubocop) 的 cops 帮助你编写更有效率的测试。

要启用它们，需要把 `test_prof/rubocop` 放到 RuboCop 的配置中：

```yml
# .rubocop.yml
require:
 - 'test_prof/rubocop'
```

根据你的需要来配置 cops：

```yml
RSpec/AggregateExamples:
  AddAggregateFailuresMetadata: false
```

或者你也可以动态地 require 它：

```sh
bundle exec rubocop -r 'test_prof/rubocop' --only RSpec/AggregateExamples
```

## RSpec/AggregateExamples

这个 cop 鼓励你使用近期 RSpec 一个极棒的特性——对测试用例内的失败进行聚合。

不用每个断言都编写一个用例，你可以对_独立_断言一起分组，这样运行所有的 setup hoods 仅需一次。
这就急剧地提升了你的性能（通过减少测试用例的总数）。

考虑这个范例：

```ruby
# bad
it { is_expected.to be_success }
it { is_expected.to have_header("X-TOTAL-PAGES", 10) }
it { is_expected.to have_header("X-NEXT-PAGE", 2) }
its(:status) { is_expected.to eq(200) }

# good
it "returns the second page", :aggregate_failures do
  is_expected.to be_success
  is_expected.to have_header("X-TOTAL-PAGES", 10)
  is_expected.to have_header("X-NEXT-PAGE", 2)
  expect(subject.status).to eq(200)
end
```

自动纠正会一般把 `:aggregate_failures` 添加到测试用例，但如果你的项目全局启用，或通过本地文件获取 meta 数据来选择性地启用，你可以使用 `AddAggregateFailuresMetadata` 配置选项来选择不添加它。

该 cop 支持自动纠正的特性，所以你可以自动重构遗留测试！

**注意**：这里展示的`its` 用例在 RSpec 3 中已经被抛弃了，但 [rspec-its gem](https://github.com/rspec/rspec-its) 的用户可以通过该 cop 来消除那种依赖。

**注意**：对于使用了 matchers 代码块的测试用例的自动纠正，比如 `change` 是故意不被支持的。