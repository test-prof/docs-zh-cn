# FactoryDefault

_Factory Default_ 旨在通过重用关联关系数据记录帮你对付 _factory cascades_（参看 [FactoryProf](../profilers/factory_prof.md)）。

**注意**，仅可用于 FactoryGirl/FactoryBot。

当你在处理典型的 SaaS 应用（或其他分等级的数据）时，它会很有用。

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

而我们想要测试 `Task` model：

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

每个测试用例会创建多少个 users 和 accounts？ 分别是两个和四个。

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

这是能工作。然而有一些缺点：有点太啰嗦了，且有错误的倾向（容易忘记某些东西）。

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

**注意**，该特性引入了一些_魔法_到你的测试中，请谨慎使用。（因为测试应该首先要易于人们阅读）。好的做法是仅用于 top-level（比如多租户中 App 中的租户）。

## 教学

在 `spec_helper.rb` 中：

```ruby
require "test_prof/recipes/rspec/factory_default"
```

这为 FactoryBot 添加了两个新方法：

- `FactoryBot#set_factory_default(factory, object)`——对于 `factory` 构建的关联关系，使用 `object` 作为默认值。

范例：

```ruby
let(:user) { create(:user) }

before { FactoryBot.set_factory_default(:user, user) }
```

- `FactoryBot#create_default(factory, *args)` —— `create` + `set_factory_default`的快捷方式。

**注意**，Defaults 默认是**在每个测试用例后被清除掉**。这意味着你在 `before(:all)` / [`before_all`](./before_all.md) / [`let_it_be`](./let_it_be.md) 定义的内部不能创建默认值。这在未来可能会改变，目前 [请查看这个变通办法](https://github.com/test-prof/test-prof/issues/125#issuecomment-471706752)。

### 与 traits 一起使用

当你在关联关系中有类似这样的 traits 时：

```ruby
factory :post do
  association :user, factory: %i[user able_to_post]
end

factory :view do
  association :user, factory: %i[user unable_to_post_only_view]
end
```

并为 `user` factory 设置了默认值——那么你会发现同样的对象被用于上面所有 factories 中。有时候这会破坏你的逻辑。

要防止这个——请设置 `FactoryDefault.preserve_traits = true` 或者使用 per-factory 来覆盖
`create_default(:user, preserve_traits: true)`。 这会针对有明确定义的 traits 的关联关系恢复到初始的  FactoryBot 行为。