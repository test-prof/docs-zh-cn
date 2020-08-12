# Any Fixture

Fixtures 是提高你测试性能的上佳方式，但对于大型项目，它们非常难于维护。

我们提出了一个更通用的方案来为你的测试套件 lazy 生成 _全局_ 状态—— AnyFixture。

有了 AnyFixture，对于数据生成你可以使用任何代码块，并且它会在测试运行结束时对其进行清理。

考虑如下范例：

```ruby
# The best way to use AnyFixture is through RSpec shared contexts
RSpec.shared_context "account", account: true do
  # You should call AnyFixture outside of transaction to re-use the same
  # data between examples
  before(:all) do
    # The provided name ("account") should be unique.
    @account = TestProf::AnyFixture.register(:account) do
      # Do anything here, AnyFixture keeps track of affected DB tables
      # For example, you can use factories here
      FactoryGirl.create(:account)

      # or with Fabrication
      Fabricate(:account)

      # or with plain old AR
      Account.create!(name: "test")
    end
  end

  # Use .register here to track the usage stats (see below)
  let(:account) { TestProf::AnyFixture.register(:account) }

  # Or hard-reload object if there is chance of in-place modification
  let(:account) { Account.find(TestProf::AnyFixture.register(:account).id) }
end

# Then in your tests

# Active this fixture using a tag
describe UsersController, :account do
  # ...
end

# This test also uses the same account record,
# no double-creation
describe PostsController, :account do
  # ...
end
```

这儿有一个现实的 [例子](http://bit.ly/any-fixture)。

## 教学

### RSpec

在 `spec_helper.rb`（或 `rails_helper.rb`，如果有的话）中：

```ruby
require "test_prof/recipes/rspec/any_fixture"
```

现在你可以在测试中使用 `TestProf::AnyFixture` 了。

### Minitest

当把 AnyFixture 与 Minitest 一起使用时，你要自己负责在每个测试运行之后清理数据库。例如：

```ruby
# test_helper.rb

require "test_prof/any_fixture"

at_exit { TestProf::AnyFixture.clean }
```

## DSL

我们提供了一个可选的 _句法糖_ （通过 Refinement）使定义 fixtures 变得更容易：

```ruby
require "test_prof/any_fixture/dsl"

# Enable DSL
using TestProf::AnyFixture::DSL

# and then you can use `fixture` method (which is just an alias for `TestProf::AnyFixture.register`)
before(:all) { fixture(:account) }

# You can also use it to fetch the record (instead of storing it in instance variable)
let(:account) { fixture(:account) }
```

## `ActiveRecord#refind`

TestProf 也提供了一个扩展来 _硬重载_ ActiveRecord 对象：

```ruby
# instead of
let(:account) { Account.find(fixture(:account).id) }

# load refinement
require "test_prof/ext/active_record_refind"

using TestProf::Ext::ActiveRecordRefind

let(:account) { fixture(:account).refind }
```

## 临时禁用 fixtures

你的有些测试可能依赖着 _清理数据库_，因此把它们跟依赖 AnyFixture 的测试一起运行就可能造成失败。

你可以在运行一个特定测试用例或用例组时，使用 `:with_clean_fixture` 的共享 context 以禁用（或删除）所有创建的 fixture：

```ruby
context "global state", :with_clean_fixture do
  # or include explicitly
  # include_context "any_fixture:clean"

  specify "table is empty or smth like this" do
    # ...
  end
end
```

这是如何工作的？它把测试用例组包裹在一个事务内（使用 [`before_all`](./before_all.md)）并在运行测试用例之前调用 `TestProf::AnyFixture.clean`。

因此，这个 context 有那么一点儿 _重_。尽量避免这些情况并编写独立于全局状态的 specs。

## 使用报告

`AnyFixture` 在测试运行期间收集了使用信息，所以可以在结束时做出报告：

```sh
[TEST PROF INFO] AnyFixture usage stats:

       key    build time  hit count    saved time

      user     00:00.004          4     00:00.017
      post     00:00.002          1     00:00.002

Total time spent: 00:00.006
Total time saved: 00:00.019
Total time wasted: 00:00.000
```

报告默认是关闭的，要启用可设置 `TestProf::AnyFixture.reporting_enabled = true` （或者可以通过`TestProf::AnyFixture.report_stats`手动执行它）。

你也可以通过 `ANYFIXTURE_REPORT=1` 环境变量来启用它。

## 警告

**更新：** 自 v0.5.0 版本起，AnyFixture 禁用了引用完整性（如果可能）以防止出现以下问题。

`AnyFixture` 以跟其填充顺序相反的顺序来清理表。这意味着当你注册一个 fixture，它引用了一个尚未注册的表的时候，一个外键的违规错误就 _可能_ 发生（如果有的话）。一个例子胜过千言万语：

```ruby
class Author < ApplicationRecord
  has_many :articles
end

class Article < ApplicationRecord
  belongs_to :author
end
```

这是共享 contexts：

```ruby
RSpec.shared_context "author" do
  before(:all) do
    @author = TestProf::AnyFixture.register(:author) do
      FactoryGirl.create(:account)
    end
  end

  let(:author) { @author }
end

RSpec.shared_context "article" do
  before(:all) do
    # outside of AnyFixture, we don't know about its dependent tables
    author = FactoryGirl.create(:author)

    @article = TestProf::AnyFixture.register(:article) do
      FactoryGirl.create(:article, author: author)
    end
  end

  let(:article) { @article }
end
```

然后在某些测试用例中：

```ruby
# This one adds only the 'articles' table to the list of affected tables
include_context "article"
# And this one adds the 'authors' table
include_context "author"
```

现在我们有这些受影响的表：`['articles', 'authors']`。在测试套件最后，'authors' 表被首先清理，这就导致了一个外键违规错误。
