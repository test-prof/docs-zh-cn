# Active Record 共享连接

**注意：** 自 Rails 5.1 起 (查看 [PR](https://github.com/rails/rails/pull/28083))已经添加了一个类似的功能。你不应该在现代 Rails 上使用 `ActiveRecordSharedConnection` ，它可能导致意料之外的行为（比如，互斥锁的死锁）。

Active Record 默认是每个线程创建一个连接。

这就不允许我们在系统测试（用 Capybara）使用 `transactional_tests` 功能（因为 Capybara 是在一个单独线程内运行 Web 服务器）。

一个通用方案是使用 `database_cleaner` 并以不带事务的策略（`truncation` / `deletion`）。但这个 _cleaning_ phase 可能会影响测试运行时间（通常如此）。

在线程之间共享连接将使我们能够像往常一样使用事务测试。

## 教学

在 `spec_helper.rb` （或 `rails_helper.rb`，如果有的话）中：

```ruby
require "test_prof/recipes/active_record_shared_connection"
```

这就自动启用了 _共享连接_ 模式。

你也可以手动启用/禁用它：

```ruby
TestProf::ActiveRecordSharedConnection.enable!
TestProf::ActiveRecordSharedConnection.disable!
```
