# FactoryAllStub

_Factory All Stub_ 是一个强制 FactoryBot/FactoryGirl 仅使用 `build_stubbed` 策略的说法（即使你调用了 `create` 或 `build`）。

背后的思想是快速修复 [Factory Doctor](../profilers/factory_doctor.md) 的问题（并甚至自动执行该操作）。

**注意**，仅支持 FactoryGirl/FactoryBot。应该仅被视为一种 specs 的临时修复。

## 教学

首先，你必须初始化 `FactoryAllStub`：

```ruby
TestProf::FactoryAllStub.init
```

初始化过程注入自定义逻辑到 FactoryBot 生成器内部。

启用 _all-stub_ 模式：

```ruby
TestProf::FactoryAllStub.enable!
```

禁用 _all-stub_ 模式并总是使用 factories：

```ruby
TestProf::FactoryAllStub.enable!
```

## RSpec

在 `spec_helper.rb` 中（或者 `rails_helper.rb` 中，如果有的话）：

```ruby
require "test_prof/recipes/rspec/factory_all_stub"
```

这会自动初始化 `FactoryAllStub` （无需调用 `.init`）并提供
`"factory:stub"` 的 shared context，用来对所标记的测试用例或测试组启用：

```ruby
describe "User" do
  let(:user) { create(:user) }

  it "is valid", factory: :stub do
    # use `build_stubbed` instead of `create`
    expect(user).to be_valid
  end
end
```

`FactoryAllStub` 被设计跟 `FactoryDoctor` 以如下方式一起使用：

```sh
# Run FactoryDoctor and mark all offensive examples with factory:stub
FDOC=1 FDOC_STAMP=factory:stub rspec ./spec/models
```
