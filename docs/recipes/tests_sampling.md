# 测试抽样

有时候在随机选择的测试上运行分析器很有用。不幸的是，测试框架不支持这种功能。这就是我们在 TestProf 中包含针对 RSpec 和 Minitest 的该小补丁的原因。

## 教学

Require 相应的补丁：

```ruby
# For RSpec in your spec_helper.rb
require "test_prof/recipes/rspec/sample"

# For Minitest in your test_helper.rb
require "test_prof/recipes/minitest/sample"
```

然后只要添加环境变量 `SAMPLE` 带上你期望的抽样数字即可：

```sh
SAMPLE=10 rspec
```

你也可以使用 `SAMPLE_GROUPS` 变量来运行随机的测试用例组（或者 suites）：

```sh
SAMPLE_GROUPS=10 rspec
```

注意，你可以把测试抽样与 RSpec filters 一起使用：

```sh
SAMPLE=10 rspec --tag=api
SAMPLE_GROUPS=10 rspec -e api
```

完成。享受吧！
