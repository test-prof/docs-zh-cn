# FactoryDefault

_FactoryDefault_ 旨在通过重用关联关系数据帮你对付 _factory cascades_（参看 [FactoryProf](../profilers/factory_prof.md)）。

当你在处理典型的 SaaS 应用（或其他分层数据）时，它会很有用。

考虑一个例子。假设你有如下的 factories：

```ruby
factory :account do
end

factory :user do
  account
end

factory :project do
  account
  user
end

factory :task do
  account
  project
  user
end
```

或者在 Fabrication 的情况下：

```
Fabricator(:account) do
end

Fabricator(:user) do
  account
end

# etc.
```

我们想要测试 `Task` model：

```ruby
describe "PATCH #update" do
  let(:task) { create(:task) }

  it "works" do
    patch :update, id: task.id, task: {completed: "t"}
    expect(response).to be_success
  end

  # ...
end
```

每个测试用例会创建多少个 users 和 accounts？ 分别是 2 和 4。

而且它破坏了我们的逻辑（每个对象应该属于同一个 account）。

典型的变通办法是：

```ruby
describe "PATCH #update" do
  let(:account) { create(:account) }
  let(:project) { create(:project, account: account) }
  let(:task) { create(:task, project: project, account: account) }

  it "works" do
    patch :update, id: task.id, task: {completed: "t"}
    expect(response).to be_success
  end
end
```

这是能工作，但有些缺点：有点太啰嗦了，且易于出错（很容易忘记某些东西）。

下面是我们如何使用 FactoryDefault 来处理它：

```ruby
describe "PATCH #update" do
  let(:account) { create_default(:account) }
  let(:project) { create_default(:project) }
  let(:task) { create(:task) }

  # and if we need more projects, users, tasks with the same parent record,
  # we just write
  let(:another_project) { create(:project) } # uses the same account
  let(:another_task) { create(:task) } # uses the same account

  it "works" do
    patch :update, id: task.id, task: {completed: "t"}
    expect(response).to be_success
  end
end
```

**注意**，该特性引入了一些_魔法_到你的测试中，请谨慎使用。（因为测试应该首先要易于人们阅读）。好仅用于 top-level（比如多租户中 App 中的租户）。

## 教学

在 `spec_helper.rb` 中：

```ruby
require "test_prof/recipes/rspec/factory_default"
```

这会将以下方法添加到 FactoryBot 和/或 Fabrication：

- `FactoryBot#set_factory_default(factory, object)` / `Fabricate.set_fabricate_default(factory, object)`——对于 `factory` 构建的关联关系，使用 `object` 作为默认值。

范例：

```ruby
let(:user) { create(:user) }

before { FactoryBot.set_factory_default(:user, user) }

# You can also set the default factory with traits
FactoryBot.set_factory_default([:user, :admin], admin)

# Or (since v1.4)
FactoryBot.set_factory_default(:user, :admin, admin)

# You can also register a default record for specific attribute overrides
Fabricate.set_fabricate_default(:post, post, state: "draft")
```

- `FactoryBot#create_default(...)` / `Fabricate.create_default(...)` – 是`create` + `set_factory_default` 的快捷方式。
- `FactoryBot#get_factory_default(factory)` / `Fabricate.get_fabricate_default(factory)` —— 获取 `factory` 的默认值（自 v1.4 起）。

```
# This method also supports traits
admin = FactoryBot.get_factory_default(:user, :admin)
```

**重要提醒：** 默认情况下， **在每个测试用例后都会清理**（即使用 `test_prof/recipes/rspec/factory_default` 时）。

### 与 `before_all` / `let_it_be` 使用

在 `before_all` 和 `let_it_be` 中创建的默认值不会在每个测试用例后重置，而只会在相应用例组后重置。因此，可以在 `let_it_be` 中调用 `create_default`，而无需任何其他配置。 **仅限 RSpec**。
**重要提醒：** 你必须在加载 BeforeAll 后加载 FactoryDefault 才能使此功能正常工作。
**注意**， 不支持常规的 `before(:all)` 回调。

### 与 traits 使用

你可以在关联中使用 traits，例如：

```ruby
factory :comment do
  user
end

factory :post do
  association :user, factory: %i[user able_to_post]
end

factory :view do
  association :user, factory: %i[user unable_to_post_only_view]
end
```

如果 `user` factory 有默认值，则它将独立于 trait 使用。这可能会破坏你的逻辑。

为防止这种情况，请配置 FactoryDefault 以保留 traits：

```ruby
# Globally
TestProf::FactoryDefault.configure do |config|
  config.preserve_traits = true
end

# or in-place
create_default(:user, preserve_traits: true)
```

使用 trait 创建默认值的工作原理如下：

```ruby
# Create a default with trait
user = create_default(:user_poster, :able_to_post)

# When an association has no traits specified, the default with trait is used
create(:comment).user == user #=> true
# When an association has the matching trait specified, the default is used, too
create(:post).user == user #=> true
# When the association's trait differs, default is skipped
create(:view).user == user #=> false
```

### 处理属性 overrides

可以为关联定义属性 overrides：

```ruby
factory :post do
  association :user, name: "Poster"
end

factory :view do
  association :user, name: "Viewer"
end
```

FactoryDefault 忽略此类 overrides，并且仍然返回默认`user`记录（如果已创建）。你可以打开属性感知功能，以便在 overrides 与默认对象属性不匹配时跳过默认记录：

```ruby
# Globally
TestProf::FactoryDefault.configure do |config|
  config.preserve_attributes = true
end

# or in-place
create_default :user, preserve_attributes: true
```

**注意：** 在未来版本的 Test Prof 中，`preserve_traits` 和 `preserve_attributes` 都将默认为 true。如果你刚开始使用此功能，我们建议将它们设置为 true。

### 忽略 default factories

可以通过使用 `skip_factory_default` 方法包装代码来临时禁用 defaults 用法：

```ruby
account = create_default(:account)
another_account = skip_factory_default { create(:account) }

expect(another_account).not_to eq(account)
```

### 显示使用统计

可以通过设置 `FACTORY_DEFAULT_SUMMARY=1` 或 `FACTORY_DEFAULT_STATS=1` 环境变量或通过设置配置值来显示 FactoryDefault 使用统计：

```ruby
TestProf::FactoryDefault.configure do |config|
  config.report_summary = true
  # Report stats prints the detailed usage information (including summary)
  config.report_stats = true
end
```

例如：

```
$ FACTORY_DEFAULT_SUMMARY=1 bundle exec rspec

FactoryDefault summary: hit=11 miss=3
```

其中 `hit` 表示在创建关联时使用 default factory 值而非新值的次数；`miss` 指示由于 traits 或属性不匹配而忽略 default 值的次数。

## Factory Default 分析，或何时使用 defaults

Factory Default 随分析器一起提供，它可以帮助查看关联在测试套件中的使用情况，以便你决定是否使用 `create_default`。

要启用性能分析，请使用 `FACTORY_DEFAULT_PROF=1` 运行测试：

```
$ FACTORY_DEFAULT_PROF=1 bundle exec rspec spec/some/file_spec.rb

.....

[TEST PROF INFO] Factory associations usage:

               factory      count    total time

                  user         17     00:42.010
         user[traited]         15     00:31.560
  user{tag:"some tag"}          1     00:00.205

Total associations created: 33
Total uniq associations created: 3
Total time spent: 01:13.775
```

由于 default factories 通常是按测试用例组（或测试类）注册的，因此我们建议针对特定文件运行此分析器，以便你可以快速识别添加 `create_default` 的可能性并提高测试速度。

**注意：** 你还可以使用分析器来测量添加 `create_default` 的效果；为此，比较在启用和禁用 FactoryDefault 的情况下运行分析器的结果（你可以通过传递 `FACTORY_DEFAULT_DISABLED=1` 环境变量来实现）。