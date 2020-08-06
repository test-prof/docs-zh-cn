# RSpecStamp

RSpecStamp 是一个自动为失败测试用例标记上自定义 tag 的工具。

它_在字面上_给你的用例加上 tag（比如，重写它们）。

RSpecStamp 的主要目标是使测试代码库重构变得容易。修改全局配置可能造成许多测试失败。你可以通过添加 shared context 为失败的 spec 打上补丁。这里 RSpecStamp 就派上用场了。

## 范例场景：Sidekiq Inline

使用 `Sidekiq::Testing.inline!` 由于其不良的性能影响可以被看作是一种 _差_ 的代码实践（原因见此 [here](https://github.com/mperham/sidekiq/issues/3495)）。但其仍然被广泛使用着。

如何从 `inline!` 迁移到 `fake!`？

第 0 步，确保你的全部测试都是通过的。

第 1 步，创建一个 shared context，有条件地打开 `inline!` 模式：

```ruby
shared_context "sidekiq:inline", sidekiq: :inline do
  around(:each) { |ex| Sidekiq::Testing.inline!(&ex) }
end
```

第 2 步，全局打开 `fake!` 模式。

第 3 步，运行 `RSTAMP=sidekiq:inline rspec`。

上面命令的输出内容包含了 _stamping_ 过程的信息：

- 有多少文件被影响到？

- 打了多少补丁？

- 多少补丁失败了？

- 有多少文件被忽略了？

现在所有（或几乎所有）失败的 specs 都被打上 `sidekiq: :inline`的 tag 了。再次运行整个测试套件，检查是否还有遗留第失败测试。

也有一种 `dry-run` 模式（通过 `RSTAMP_DRY_RUN=1` 环境变量激活），打印出补丁内容而不是直接重写文件。

## 配置

默认情况下，RSpecStamp 忽略 `spec/support` 目录下的用例（放置 shared 用例的典型位置）。
你可以添加更多的 _忽略_ 模式：

```ruby
TestProf::RSpecStamp.configure do |config|
  config.ignore_files << %r{spec/my_directory}
end
```
