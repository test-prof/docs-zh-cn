# 详细日志

有时侯挖掘日志是找出正在发生的事情的最佳方法。

当你运行测试套件时，默认是不打印日志出来的（尽管写到了 `test.log`——谁会去看呢？）

我们提供了一个配方以对特定的测试用例/组来开启详细日志

**注意：** 仅支持 Rails。

## 教学

把下面这行放入你的 `rails_helper.rb` / `spec_helper.rb` / `test_helper.rb`，哪个都行：

```ruby
require "test_prof/recipes/logging"
```

### Log everything

使用环境变量 `LOG` 来启用全局日志记录：

```sh
# log everything to stdout
LOG=all rspec ...

# or
LOG=all rake test

# log only Active Record statements
LOG=ar rspec ...
```

### Per-example logging

**注意：** 仅支持 RSpec。

通过添加特别 tag —— `:log`来激活日志记录：

```ruby
# Add the tag and you will see a lot of interesting stuff in your console
it "does smthng weird", :log do
  # ...
end

# or for the group
describe "GET #index", :log do
  # ...
end
```

要只对 Active Record 启用，使用 `log: :ar` tag：

```ruby
describe "GET #index", log: :ar do
  # ...
end
```

### Logging helpers

对于更多粒度的控制，你可以使用 `with_logging` （记录一切）和
`with_ar_logging` （记录 Active Record）两个帮助方法：

```ruby
it "does somthing" do
  do_smth
  # show logs only for the code within the block
  with_logging do
    # ...
  end
end
```

**注意：** 要在 Minitest 使用该帮助方法，你应该手动包含 `TestProf::Rails::LoggingHelpers` 模块：

```ruby
class MyLoggingTest < Minitest::Test
  include TestProf::Rails::LoggingHelpers
end
```
