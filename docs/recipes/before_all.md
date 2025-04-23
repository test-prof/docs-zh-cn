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

但然后你就必须处理数据库的清理，这要么棘手要么很慢。

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

就这样！

**注意**：需要 RSpec >= 3.3.0。

**注意**：`before_all` 所提供的强大能力要求你承担相应巨大的责任。请务必查看 [Caveats section](#caveats) 文档的具体细节。

## 教学

### 多数据库支持

ActiveRecord BeforeAll 适配器将仅使用 ActiveRecord::Base 连接来启动事务。如果要确保 `before_all` 可以使用多个连接，则需要确保在使用 `before_all` 之前加载连接类。

例如，假设你有 `ApplicationRecord` 和一个单独的 user accounts 数据库：

```ruby
class Users < AccountsRecord
  # ...
end

class Articles < ApplicationRecord
  # ...
end
```

那么，在运行测试之前，确实需要加载这两个 Connection Classes：

```ruby
# Ensure connection classes are loaded
ApplicationRecord
AccountsRecord
```

此代码可以添加到 `rails_helper.rb` 或运行 minitests 的 rake 任务中。

### RSpec

在你的 `rails_helper.rb`中（或 `spec_helper.rb` 中的 *ActiveRecord* 被加载之后）：

```ruby
require "test_prof/recipes/rspec/before_all"
```

**注意**：`before_all` （和依赖它的 `let_it_be`），不会把单独的测试包裹在它自己的数据库事务中。请使用 Rails 原生的 `use_transactional_tests`
（在 Rails < 5.1 中是 `use_transactional_fixtures` ）、RSpec Rails 的`use_transactional_fixtures`、
DatabaseCleaner 或者在每个测试用例前开始事务并在结束后回滚的自定义代码。

### Minitest

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

除了`before_all`，TestProf 也提供了一个`after_all`回调，它在由`before_all`所打开的数据库事务关闭之前被调用，即在测试类的最后一个用例完成之后。

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

你可以注册回调以在 `before_all` 打开和回滚事务之前/之后来运行：

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

### 数据库回滚到原始状态，但 objects 并非如此。

如果你在测试用例中修改了 `before_all` 代码块中所生成的 objects，那么可能需要重新初始化它们：

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

解决这个问题最简单的方式是每个测试用例都重新加载记录（这_仍然_比创建新的要快得多）：

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

### 数据库在测试之间不会回滚

数据库在 RSpec 测试用例之间不会回滚，仅在测试用例组之间回滚。我们不想重新发明轮子，所以鼓励你使用其他自带支持该功能的工具。

如果你使用 RSpec Rails，在你的 `spec/rails_helper.rb`中打开`RSpec.configuration.use_transactional_fixtures`：

```ruby
RSpec.configure do |config|
  config.use_transactional_fixtures = true # RSpec takes care to use `use_transactional_tests` or `use_transactional_fixtures` depending on the Rails version used
end
```

如果你使用 Minitest，请确认设置了 `use_transactional_tests`（在 Rails < 5.1 中是`use_transactional_fixtures` ）为 `true` 。

如果你在使用 DatabaseCleaner，请确保它是在测试之间进行回滚。

## 与 Isolator 一起使用

[Isolator](https://github.com/palkan/isolator) 是一个数据库事务内潜在违反原子性的运行时检测器（比如，进行 HTTP 调用或者后台队列作业）。

TestProf 自带对 Isolator 的识别并使其忽略 `before_all` 的事务。

你只需确保在加载 `before_all`（或 `let_it_be`）之前 require `isolator`。

或者，你也可以显式地加载该补丁：

```ruby
# after loading before_all or/and let_it_be
require "test_prof/before_all/isolator"
```

## 使用 Rails fixtures (_实验性的_)

如果你想要在`before_all` hook 内使用 fixture，必须通过`setup_fixture:`选项来明确加入：

```ruby
before_all(setup_fixtures: true) do
  @user = users(:john)
  @post = create(:post, user: user)
end
```

Minitest 和 RSpec 都可用。

你也可以全局启用 fixtures（即对所有`before_all` hooks）:

```ruby
TestProf::BeforeAll.configure do |config|
  config.setup_fixtures = true
end
```

## 全局 Tags

你可以使用 tags 为特定 RSpec 测试用例组注册回调：

```ruby
TestProf::BeforeAll.configure do |config|
  config.before(:begin, reset_sequences: true, foo: :bar) do
    warn <<~MESSAGE
      Do NOT create objects outside of transaction
      because all db sequences will be reset to 1
      in every single example, so that IDs of new objects
      can get into conflict with the long-living ones.
    MESSAGE
  end
end
```

