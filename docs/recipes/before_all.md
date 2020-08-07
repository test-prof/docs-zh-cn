# Before All

Rails 有一个很棒的特性——`transactional_tests`，比如，在一个事务之内运行每个测试用例，该用例会在结束时回滚。

这样就没有测试用例污染全局数据库状态了。

但如果在一个通用 setup 有许多用例的情况呢？

当然，我们可以象这样来做：

```ruby
describe BeatleWeightedSearchQuery do
  before(:each) do
    @paul = create(:beatle, name: "Paul")
    @ringo = create(:beatle, name: "Ringo")
    @george = create(:beatle, name: "George")
    @john = create(:beatle, name: "John")
  end

  # and about 15 examples here
end
```

或者你也可以尝试 `before(:all)`：

```ruby
describe BeatleWeightedSearchQuery do
  before(:all) do
    @paul = create(:beatle, name: "Paul")
    # ...
  end

  # ...
end
```

但然后你就不得不处理数据库的清理工作，其要么乏味要么迟缓。

有一个更好的做法：我们可以把整个测试用例组包裹在一个事务内。
而这正是 `before_all` 工作的方式：

```ruby
describe BeatleWeightedSearchQuery do
  before_all do
    @paul = create(:beatle, name: "Paul")
    # ...
  end

  # ...
end
```

就是如此！

**注意**：需要 RSpec >= 3.3.0。

**注意**：`before_all` 所提供的强大能力要求你承担相应强大的责任。
确认查看了 [Caveats section](#caveats) 该文的具体细节。

## 教学

### RSpec

在你的 `rails_helper.rb`中（或 `spec_helper.rb` 中的 *ActiveRecord* 被加载之后）：

```ruby
require "test_prof/recipes/rspec/before_all"
```

**注意**：`before_all` （和依赖它的 `let_it_be`），未把单独的测试包裹在它自己的数据库事务中。请使用 Rails 原生的 `use_transactional_tests`
（Rails < 5.1中是`use_transactional_fixtures` ），RSpec Rails 的`use_transactional_fixtures`，
DatabaseCleaner，或者在每个测试用例前开始事务并在结束后回滚的自定义代码。

### Minitest（实验性的）

\*_实验性的_ 意味着我还没有在生产环境中用过。

可以与 Minitest 一起使用 `before_all` ：

```ruby
require "test_prof/recipes/minitest/before_all"

class MyBeatlesTest < Minitest::Test
  include TestProf::BeforeAll::Minitest

  before_all do
    @paul = create(:beatle, name: "Paul")
    @ringo = create(:beatle, name: "Ringo")
    @george = create(:beatle, name: "George")
    @john = create(:beatle, name: "John")
  end

  # define tests which could access the object defined within `before_all`
end
```

## 数据库适配器

你可以不只跟 ActiveRecord（开箱即用）也可跟其他数据库工具一起使用 `before_all`。

你需要做的就是构建一个自定义适配器并配置 `before_all` 使用它：

```ruby
class MyDBAdapter
  # before_all adapters must implement two methods:
  # - begin_transaction
  # - rollback_transaction
  def begin_transaction
    # ...
  end

  def rollback_transaction
    # ...
  end
end

# And then set adapter for `BeforeAll` module
TestProf::BeforeAll.adapter = MyDBAdapter.new
```

## Hooks

> 自 v0.9.0 起

你可以在 `before_all` 打开和回滚一个事务之前/之后注册回调来运行它：

```ruby
TestProf::BeforeAll.configure do |config|
  config.before(:begin) do
    # do something before transaction opens
  end
  # after(:begin) is also available

  config.after(:rollback) do
    # do something after transaction closes
  end
  # before(:rollback) is also available
end
```

请查看 [Discourse](https://github.com/discourse/discourse/blob/4a1755b78092d198680c2fe8f402f236f476e132/spec/rails_helper.rb#L81-L141) 中的范例。

## 警告

### 数据库是被回滚到全新的初始状态，但对象并非如此。

如果你在测试用例中更改了 `before_all` 代码块中所生成的对象，那么可能不得不重新初始化它们：

```ruby
before_all do
  @user = create(:user)
end

let(:user) { @user }

it "when user is admin" do
  # we modified our object in-place!
  user.update!(role: 1)
  expect(user).to be_admin
end

it "when user is regular" do
  # now @user's state depends on the order of specs!
  expect(user).not_to be_admin
end
```

解决这个问题最容易的方式是每个测试用例都重新加载记录（这_仍然_比创建一个新的要快得多）：

```ruby
before_all do
  @user = create(:user)
end

# Note, that @user.reload may not be enough,
# 'cause it doesn't reset associations
let(:user) { User.find(@user.id) }

# or with Minitest
def setup
  @user = User.find(@user.id)
end
```

### 数据库在测试之间没有回滚

数据库在 RSpec 测试用例之间不回滚，仅在测试用例组之间回滚。
我们不想重新发明轮子，所以鼓励你使用其他自带支持该功能的工具。

如果你使用 RSpec Rails，在你的 `spec/rails_helper.rb`中打开`RSpec.configuration.use_transactional_fixtures`：

```ruby
RSpec.configure do |config|
  config.use_transactional_fixtures = true # RSpec takes care to use `use_transactional_tests` or `use_transactional_fixtures` depending on the Rails version used
end
```

如果你使用 Minitest，请确认设置了 `use_transactional_tests`（在 Rails < 5.1 中是`use_transactional_fixtures` ）为 `true` 。

如果你在使用 DatabaseCleaner，请确认其是在测试之间进行回滚。

## 与 Isolator 一起使用的方法

[Isolator](https://github.com/palkan/isolator) 是一个数据库事务内潜在违反原子性的运行时检测器（比如，进行 HTTP 调用或者后台队列作业）。

TestProf 自带对 Isolator 的识别并使其忽略 `before_all` 的事务。

你只需确保在加载 `before_all`（或 `let_it_be`）之前 require `isolator`。

要不然，你也可以明确地加载该补丁：

```ruby
# after loading before_all or/and let_it_be
require "test_prof/before_all/isolator"
```
