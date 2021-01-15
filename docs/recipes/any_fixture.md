# Any Fixture

Fixtures 是一种提高你测试性能的上佳方式，但对于大型项目，它们非常难于维护。

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

报告默认是关闭的，要启用可设置 `TestProf::AnyFixture.config.reporting_enabled = true` （或者可以通过`TestProf::AnyFixture.report_stats`手动执行它）。

你也可以通过 `ANYFIXTURE_REPORT=1` 环境变量来启用它。

## 使用自动生成的 SQL 导出文件

>  @since v1.0, 实验性的

AnyFixture 被设计为在每个测试套件运行时生成数据（并在结束时清理掉）。这依然有着时间的消耗（比如，对于系统或性能测试）；因此，我们想要进一步优化。

我们提供了另一种对测试数据提速的方式，称作`#register_dump`。它的工作方式类似于针对初次运行的`#register`：接收一个代码块，并追踪其内部所生成的 SQL 查询。然后，它生成一份 SQL 导出文本，表示其调用期间创建或修改的数据，并使用这份导出对随后的测试恢复数据库状态。

来看一个范例：

```ruby
RSpec.shared_context "account", account: true do
  # You should call AnyFixture outside of transaction to re-use the same
  # data between examples
  before(:all) do
    # The block is called once per test run (similary to #register)
    TestProf::AnyFixture.register_dump("account") do
      # Do anything here, AnyFixture keeps track of affected DB tables
      # For example, you can use factories here
      account = FactoryGirl.create(:account, name: "test")

      # or with Fabrication
      account = Fabricate(:account, name: "test")

      # or with plain old AR
      account = Account.create!(name: "test")

      # updates are also tracked
      account.update!(tag: "sql-dump")
    end
  end

 # Here, we MUST use a custom way to retrieve a record: since we restore the data
 # from a plain SQL dump, we have no knowledge of Ruby objects
  let(:account) { Account.find_by!(name: "test") }
end
```

而下面是运行测试时所发生的：

```bash
# first run
$ bundle exec rspec

# AnyFixture.register_dump is called:
# - is SQL dump present? No
# - run block and write all modifying queries to a new SQL dump
# AnyFixture.clean is called:
# - clean all the affected tables

# second run
$ bundle exec rspec

# AnyFixture.register_dump is called:
# - is SQL dump present? Yes
# - restore dump (do not run block)
# AnyFixture.clean is called:
# - clean all the affected tables
```

### 所需环境

目前，仅支持 PostgreSQL 12+ 和 SQLite3。

### 导出文件失效

所生成的导出文件会因多种原因而过期：数据库 schema 变更，fixture 代码块升级，等等。要处理这些无效情况，我们使用文件内容的 digests 作为缓存 keys（导出文件名为后缀）。

默认情况下，AnyFixture 会监控`db/schema.rb`，`db/structure.sql`和名为`\#register_dump`的文件。

默认监控文件列表可以通过修改`default_dump_watch_paths`配置参数来变更：

```ruby
TestProf::AnyFixture.configure do |config|
  # you can use exact file paths or globs
  config.default_dump_watch_paths << Rails.root.join("spec/factories/**/*")
end
```

另外，你也可以通过`watch`选项把监控文件添加到一个特别的`#register_dump`调用上：

```ruby
TestProf::AnyFixture.register_dump("account", watch: ["app/models/account.rb", "app/models/account/**/*,rb"]) do
  # ...
end
```

**注意**：当你使用`watch`选项时，当前文件并未被添加到监控列表里。你要使用`__FILE__`来明确指定它。

最后，如果你想要强制重新生成导出文件，可以使用`ANYFIXTURE_FORCE_DUMP`环境变量：

- `ANYFIXTURE_FORCE_DUMP=1`会强制全部导出文件都重新生成。
- `ANYFIXTURE_FORCE_DUMP=account`会强制重新生成仅匹配的导出文件（比如，匹配`/account/`的）。

#### 缓存 keys

可以提供自定义的缓存 keys，被用作 digest 的一部分：

```ruby
# cache_key could be pretty much anything that responds to #to_s
TestProf::AnyFixture.register_dump("account", cache_key: ["str", 1, {key: :val}]) do
  # ...
end
```

### Hooks

#### `before` / `after`

Before hooks 要么在调用一个 fixture 代码块 之前被调用，要么在恢复一个导出文件之前被调用。一种特别的使用场景是在多租户应用中重新创建一个租户：

```ruby
TestProf::AnyFixture.register_dump(
  "account",
  before: proc do
    begin
      Apartment::Tenant.create("test")
    rescue
      nil
    end
    Apartment::Tenant.create("test")
  end
) do
  # ...
end
```

类似地，after hooks 也是要么在调用一个 fixture 代码块 之前被调用，要么在恢复一个导出文件之前被调用。

你也能指定全局的 before 和 after hooks：

```ruby
TestProf::AnyFixture.configure do |config|
  config.before_dump do |dump:, import:|
    # dump is an object containing information about the dump (e.g., dump.digest)
    # import is true if we're restoring a dump and false otherwise
    # do something
  end

  config.after_dump do |dump:, import:|
    # ...
  end
end
```

**注意**：after 回调总是会执行，哪怕导出文件的创建失败了也会。你可以使用`dump.success?`方法来检测数据生成是否成功。

#### `skip_if`

该回调仅作为`#register_dump`的选项时可用，可被用来完全忽略 fixture。这在当你想要在测试运行之间保护数据库状态（比如，不清理数据库）的时候很有用。

下面是一个完整示例：

```ruby
TestProf::AnyFixture.register_dump(
  "account",
  # do not track tables for AnyFixture.clean (though other fixtures could affect this)
  clean: false,
  skip_if: proc do |dump:|
    Apartment::Tenant.switch!("test")
    # if the current account has matching meta — the database is in actual state
    Account.find_by!(name: "test").meta["dump-version"] == dump.digest
  end,
  before: proc do
    begin
      Apartment::Tenant.create("test")
    rescue
      nil
    end
    Apartment::Tenant.create("test")
  end,
  after: proc do |dump:, import:|
    # do not persist dump version if dump failed or we're restoring data
    next if import || !dump.success?

    Account.find_by!(name: "test").then do |account|
      account.meta["dump-version"] = dump.digest
      account.save!
    end
  end
) do
  # ...
end
```

### 配置

下面是一些可用的配置项：

```ruby
TestProf::AnyFixture.configure do |config|
  # Where to store dumps (by default, TestProf.artifact_path + '/any_dumps')
  config.dumps_dir = "any_dumps"
  # Include mathing queries into a dump (in addition to INSERT/UPDATE/DELETE queries)
  config.dump_matching_queries = /^$/
  # Whether to try using CLI tools such as psql or sqlite3 to restore dumps or not (and use ActiveRecord instead)
  config.import_dump_via_cli = false
end
```

**注意**：当使用 CLI 工具来恢复导出文件时，无法追踪影响到的数据表，并因此无法通过`AnyFixture.clean`来清理它们。


