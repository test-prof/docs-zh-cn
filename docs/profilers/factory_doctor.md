# Factory 医生

拖慢我们测试的一种常见坏模式是不必要的数据库操作。考虑一个_不好_的例子：

```ruby
# with FactoryBot/FactoryGirl
it "validates name presence" do
  user = create(:user)
  user.name = ""
  expect(user).not_to be_valid
end

# with Fabrication
it "validates name presence" do
  user = Fabricate(:user)
  user.name = ""
  expect(user).not_to be_valid
end
```

这里我们创建了一条新的 user 记录，运行所有的回调和校验，并把它保存到数据库。而我们完全不需要这样！下面是一个_好的_ 示例：

```ruby
# with FactoryBot/FactoryGirl
it "validates name presence" do
  user = build_stubbed(:user)
  user.name = ""
  expect(user).not_to be_valid
end

# with Fabrication
it "validates name presence" do
  user = Fabricate.build(:user)
  user.name = ""
  expect(user).not_to be_valid
end
```

查看更多有关 [`build_stubbed`](https://robots.thoughtbot.com/use-factory-girls-build-stubbed-for-a-faster-test) 的内容。

FactoryDoctor 是一个帮你识别这些_坏_测试的工具，比如，测试执行了不必要的数据库查询。

输出示例：

```sh
[TEST PROF INFO] FactoryDoctor report

Total (potentially) bad examples: 2
Total wasted time: 00:13.165

User (./spec/models/user_spec.rb:3) (3 records created, 00:00.628)
  validates name (./spec/user_spec.rb:8) – 1 record created, 00:00.114
  validates email (./spec/user_spec.rb:8) – 2 records created, 00:00.514
```

**注意**：注意到“potentially” 的字样？抱歉，FactoryDoctor 并非魔术师（它仍在学习中），有时它也会发生“误诊”的事情。

如果你发现了造成 FactoryDoctor 失败的案例，请提交 [issue](https://github.com/test-prof/test-prof/issues)。

你还可以告诉 FactoryDoctor 忽略特定的测试用例/组。只要添加 `:fd_ignore` tag 即可：

```ruby
# won't be reported as offense
it "is ignored", :fd_ignore do
  user = create(:user)
  user.name = ""
  expect(user).not_to be_valid
end
```

## 教学

FactoryDoctor 支持：

- FactoryGirl/FactoryBot
- Fabrication

### RSpec

使用 `FDOC` 环境变量来激活 FactoryDoctor：

```sh
FDOC=1 rspec ...
```

### 与 RSpecStamp 一起使用

FactoryDoctor 可与 [RSpec Stamp](../recipes/rspec_stamp.md) 一起使用，以自定义 tag 来自动标记 _坏的_ 测试用例。例如：

```sh
FDOC=1 FDOC_STAMP="fdoc:consider" rspec ...
```

运行上面命令之后，所有_潜在的坏测试用例会都以`fdoc: :consider` tag 标记出来。

### Minitest

使用 `FDOC` 环境变量来激活 FactoryDoctor：

```sh
FDOC=1 ruby ...
```

或者使用如下 CLI 选项：

```sh
ruby ... --factory-doctor
```

强制 Factory Doctor 忽略特定测试用例的相同选项对于 Minitest 也是可用的。只要在你的测试用例内使用 `fd_ignore`：

```ruby
# won't be reported as offense
it "is ignored" do
  fd_ignore

  @user.name = ""
  refute @user.valid?
end
```

### 与 Minitest::Reporters 一起使用

如果在你的项目中正在使用 `Minitest::Reporters`，则必须在测试帮助方法文件中显式声明它：

```sh
require 'minitest/reporters'
Minitest::Reporters.use! [YOUR_FAVORITE_REPORTERS]
```

**注意**: 当你以 gem 安装了 `minitest-reporters` 但并未在你的 `Gemfile` 里声明时，
确保总是在测试运行命令之前使用 `bundle exec` （但我们相信你始终这样做）。
否则，你会得到由 Minitest plugin 系统造成的报错, 它会扫描 `$LOAD_PATH` 用于任何 `minitest/*_plugin.rb`，因此该场景中可用的 `minitest-reporters` plugin 的初始化不会正确发生。

## 配置

如下配置参数是可用的（展示的为默认情况）：

```ruby
TestProf::FactoryDoctor.configure do |config|
  # Which event to track within test example to consider them "DB-dirty"
  config.event = "sql.active_record"
  # Consider result "good" if the time in DB is less then the threshold
  config.threshold = 0.01
end
```

你也可以使用相应的环境变量： `FDOC_EVENT` 和 `FDOC_THRESHOLD`。
