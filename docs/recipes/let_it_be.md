# Let It Be

让我们带来一点魔法，介绍一种设置_shared_测试数据的新方法。

假设你有如下设定：

```ruby
describe BeatleWeightedSearchQuery do
  let!(:paul) { create(:beatle, name: "Paul") }
  let!(:ringo) { create(:beatle, name: "Ringo") }
  let!(:george) { create(:beatle, name: "George") }
  let!(:john) { create(:beatle, name: "John") }

  specify { expect(subject.call("john")).to contain_exactly(john) }

  # and more examples here
end
```

我们不需要为每个测试用例都创建这个“四人组”，对吧？

我们已经有了 [`before_all`](./before_all.md) 来解决_重复性数据_的问题：

```ruby
describe BeatleWeightedSearchQuery do
  before_all do
    @paul = create(:beatle, name: "Paul")
    # ...
  end

  specify { expect(subject.call("joh")).to contain_exactly(@john) }

  # ...
end
```

这个方法工作良好，但需要我们使用实例变量以及一次性定义所有东西。这样便不易于重构使用 `let/let!`的已有测试。

使用 `let_it_be` 你就能做到如下：

```ruby
describe BeatleWeightedSearchQuery do
  let_it_be(:paul) { create(:beatle, name: "Paul") }
  let_it_be(:ringo) { create(:beatle, name: "Ringo") }
  let_it_be(:george) { create(:beatle, name: "George") }
  let_it_be(:john) { create(:beatle, name: "John") }

  specify { expect(subject.call("john")).to contain_exactly(john) }

  # and more examples here
end
```

完成！只用把 `let!` 替换为 `let_it_be` 即可。这等价于 `before_all` 方案而只要更少的重构。

**注意**： `before_all` 提供的强大能力需要担负巨大的责任。请务必参考 [Caveats section](#caveats) 文档的具体细节。

## 教学

在 `rails_helper.rb` 或 `spec_helper.rb`中：

```ruby
require "test_prof/recipes/rspec/let_it_be"
```

在测试中：

```ruby
describe MySuperDryService do
  let_it_be(:user) { create(:user) }

  # ...
end
```

`let_it_be` 不会在测试用例之间自动把数据库恢复之前的状态，它仅在测试用例组之间这样做。
请使用 Rails 原生的 `use_transactional_tests` （在 Rails < 5.1 中是`use_transactional_fixtures` ）、RSpec Rails 的 `use_transactional_fixtures`、DatabaseCleaner 或者在每个测试用例前开始事务并在结束后回滚的自定义代码。

## 警告

### 数据库被回滚到全新的初始状态，但 objects 并非如此。

如果你在测试用例中修改了 `let_it_be` 代码块中所生成的 objects，那么可能需要重新初始化它们。我们有一个内置的_修饰器_支持这个。

### 数据库在测试之间不会回滚

数据库不会在 RSpec 测试用例之间回滚，仅在测试用例组之间回滚。我们不想重新发明轮子，所以鼓励你使用其他自带支持该功能的工具。

如果你使用 RSpec Rails，在你的 `spec/rails_helper.rb`中打开`RSpec.configuration.use_transactional_fixtures`：

```ruby
RSpec.configure do |config|
  config.use_transactional_fixtures = true # RSpec takes care to use `use_transactional_tests` or `use_transactional_fixtures` depending on the Rails version used
end
```

如果你使用 Minitest，请确认设置了 `use_transactional_tests`（在 Rails < 5.1 中是`use_transactional_fixtures` ）为 `true` 。

如果你在使用 DatabaseCleaner，请确认其是在测试之间进行回滚。

## Aliases

命名是很难的。处理边界场景（上述情况）也不棘手。

为了解决这个问题，我们提供了一种方式以预定义选项来定义 `let_it_be` 的 aliases：

```ruby
# rails_helper.rb
TestProf::LetItBe.configure do |config|
  # define an alias with `refind: true` by default
  config.alias_to :let_it_be_with_refind, refind: true
end

# then use it in your tests
describe "smth" do
  let_it_be_with_refind(:foo) { Foo.create }

  # refind can still be overridden
  let_it_be_with_refind(:bar, refind: false) { Bar.create }
end
```

## 修饰器

如果你修改了测试用例内的 `let_it_be` 代码块中所生成的 objects，那么可能需要重新初始化它们以避免在测试用例之间的状态泄漏。请记住，即使数据库能被回滚到原始状态，而 models 本身也不会。

我们有一个内置的 _修饰器_ 支持把 models 恢复到原始状态：

```ruby
# Use reload: true option to reload user object (assuming it's an instance of ActiveRecord)
# for every example
let_it_be(:user, reload: true) { create(:user) }

# it is almost equal to
before_all { @user = create(:user) }
let(:user) { @user.reload }

# You can also specify refind: true option to hard-reload the record
let_it_be(:user, refind: true) { create(:user) }

# it is almost equal to
before_all { @user = create(:user) }
let(:user) { User.find(@user.id) }
```

**注意：** 请确保 require `let_it_be` 是在 `active_record` 被加载之后（比如，在 `rails_helper.rb` 中是在 require Rails app **之后**）；否则， `refind` 和 `reload` 修饰器不会被激活。

你也可以把修饰器跟数组值一起使用了，比如， `create_list`：

```ruby
let_it_be(:posts, reload: true) { create_list(:post, 3) }

# it's the same as
before_all { @posts = create_list(:post, 3) }
let(:posts) { @posts.map(&:reload) }
```

### 自定义修饰器

如果 `reload` 和 `refind` 还不够，那你可以添加自定义修饰器：

```ruby
# rails_helper.rb
TestProf::LetItBe.configure do |config|
  # Define a block which will be called when you access a record first within an example.
  # The first argument is the pre-initialized record,
  # the second is the value of the modifier.
  #
  # This is how `reload` modifier is defined
  config.register_modifier :reload do |record, val|
    # ignore when `reload: false`
    next record unless val
    # ignore non-ActiveRecord objects
    next record unless record.is_a?(::ActiveRecord::Base)
    record.reload
  end
end
```

### 默认修饰器

对于所有 `let_it_be` 调用，可以配置其默认修饰器：

- 全局：

```ruby
TestProf::LetItBe.configure do |config|
  # Make refind activated by default
  config.default_modifiers[:refind] = true
end
```

- 对使用 tags 的特定 contexts：

```ruby
context "with let_it_be reload", let_it_be_modifiers: {reload: true} do
  # examples
end
```

**注意** 嵌套 contexts tags 是被覆盖而非合并：

```ruby
TestProf::LetItBe.configure do |config|
  config.default_modifiers[:freeze] = false
end

context "with reload", let_it_be_modifiers: {reload: true} do
  # uses freeze: false, reload: true here

  context "with freeze", let_it_be_modifiers: {freeze: true} do
    # uses only freeze: true (reload: true is overwritten by new metadata)
  end
end
```

## 状态泄漏检测

[`rspec-rails` 文档](https://relishapp.com/rspec/rspec-rails/v/3-9/docs/transactions) 对于事务和 `before(:context)`的描述：

> 即使每个测试用例中的数据库更新都将回滚，object 也不会知道这些回滚，因此该 object 与其数据很容易不同步。

由于 `let_it_be` 是在其引擎下的 `before(:context)` hooks 中初始化对象，因此会受到此问题影响：代码可能会修改测试用例之间共享的 models（因此造成_共享状态泄漏_）。这可能是非主动地：当测试下的底层代码修改了 models 时，比如，修改了 `updated_at` 属性；或者可能是有意地：当 models 在 `before` hooks 或测试用例自身中被更新，而非在初始的适当状态中被创建的时候。

这种状态泄漏带来了对其他测试用例的潜在有害的边际效应，比如隐式的依赖和执行顺序依赖。

对多个测试用例之间的多个共享 models，很难追踪这些用例和修改 model 的代码的准确位置。

为了检测到更改，可让传递给 `let_it_be` 的对象被冻结（以 `#freeze`），并且引发 `FrozenError`：

```ruby
# use freeze: true modifier to enable this feature
let_it_be(:user, freeze: true) { create(:user) }

# it is almost equal to
before_all { @user = create(:user).freeze }
let(:user) { @user }
```

要修复 `FrozenError`：

- 添加 `reload: true` 或 `refind: true`, 它可以消除泄漏检测并防止泄漏本身。通常在每个测试用例之前重新载入 model 比从头重新创建 model 要明显更快（在某些情况下要快两个甚至三个数量级）。
- 重写造成问题的测试代码。

该功能是可选的，因为它可能会发现 specs 中的大量泄漏，而一次性修复所有泄漏可能是一种重大负担。可以针对部分 specs 逐步启用它（比如，仅针对 models），如下：

```ruby
# spec/spec_helper.rb
RSpec.configure do |config|
  # ...
  config.define_derived_metadata(let_it_be_frost: true) do |metadata|
    metadata[:let_it_be_modifiers] ||= {freeze: true}
  end
end
```

然后给 contexts 或测试用例打上 `:let_it_be_frost` 的 tag 来启用该功能。

或者，你还可以显式指定 `freeze` 修饰器（`let_it_be(freeze: true)`）或者配置一个 alias。

## 报告重复项

虽然我们建议使用 `let_it_be` 而不是 `let！`，但它们有一个重要的区别：你可以用相同或嵌套的 context 覆盖 `let！` 定义，从而只调用后者；而`let_it_be` 记录虽可被覆盖，但仍会创建。例如：

```ruby
context "A" do
  let!(:user) { create(:user, name: "a") }
  let_it_be(:post) { create(:post, title: "A") }

  specify { expect(User.all.pluck(:name)).to eq ["a"] }
  specify { expect(Post.all.pluck(:title)).to eq ["A"] }

  context "B" do
    let!(:user) { create(:user, name: "b") }
    let_it_be(:post) { create(:post, title: "B") }

    specify { expect(User.all.pluck(:name)).to eq ["b"] }
    specify { expect(Post.all.pluck(:title)).to eq ["B"] } # fails, because there are two posts
  end
end
```

因此，为方便起见，你可以配置覆盖 let_it_be 时的行为。

```ruby
TestProf::LetItBe.configure do |config|
  config.report_duplicates = :warn # Rspec.warn_with
  config.report_duplicates = :raise # Kernel.raise
end
```

默认情况下，此参数处于禁用状态。你可以配置将生成警告或引发异常的行为。
