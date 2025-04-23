# 内存分析器

MemoryProf 在测试套件运行期间跟踪内存使用情况，并可以帮助检测导致内存峰值的测试用例和组。内存分析支持两个指标：RSS 和 allocations。

输出范例：

```sh
[TEST PROF INFO] MemoryProf results

Final RSS: 673KB

Top 5 groups (by RSS):

AnswersController (./spec/controllers/answers_controller_spec.rb:3) – +80KB (13.50%)
QuestionsController (./spec/controllers/questions_controller_spec.rb:3) – +32KB  (9.08%)
CommentsController (./spec/controllers/comments_controller_spec.rb:3) – +16KB (3.27%)

Top 5 examples (by RSS):

destroys question (./spec/controllers/questions_controller_spec.rb:38) – +144KB (24.38%)
change comments count (./spec/controllers/comments_controller_spec.rb:7) – +120KB (20.00%)
change Votes count (./spec/shared_examples/controllers/voted_examples.rb:23) – +90KB (16.36%)
change Votes count (./spec/shared_examples/controllers/voted_examples.rb:23) – +64KB (12.86%)
fails (./spec/shared_examples/controllers/invalid_examples.rb:3) – +32KB (5.00%)
```

examples 块显示每个示例使用的内存量，groups 块显示组中定义的其他代码分配的内存。例如，RSpec 组可能包含繁重的 `before(:all)` （或 `before_all` ）设置块，因此查看哪些组在其示例之外使用最多的内存量会很有帮助。

## 指示

使用以下方式激活 MemoryProf：

### RSpec

使用 `TEST_MEM_PROF` 环境变量设置要使用的指标：

```sh
TEST_MEM_PROF='rss' rspec ...
TEST_MEM_PROF='alloc' rake rspec ...
```

### Minitest

使用 `TEST_MEM_PROF` 环境变量设置要使用的指标：

```sh
TEST_MEM_PROF='rss' rake test
TEST_MEM_PROF='alloc' rspec ...
```

或使用 CLI 选项：

```sh
# Run a specific file using CLI option
ruby test/my_super_test.rb --mem-prof=rss

# Show the list of possible options:
ruby test/my_super_test.rb --help
```

## 配置

默认情况下，MemoryProf 会跟踪使用最大内存量的前 5 个示例和组。你可以使用以下选项设置要显示的示例/组数：

```sh
TEST_MEM_PROF='rss' TEST_MEM_PROF_COUNT=10 rspec ...
```

或使用 Minitest 的 CLI 选项：

```sh
# Run a specific file using CLI option
ruby test/my_super_test.rb --mem-prof=rs --mem-prof-top-count=10
```

## 支持的 Ruby 引擎和操作系统

目前 JRuby 不支持 allocation 模式。

由于 RSS 依赖于操作系统，因此 MemoryProf 使用不同的工具来检索它：

* Linux – `/proc/$pid/statm` 文件
* macOS, Solaris, BSD – `ps`,
* Windows – `Get-Process`, 需要安装 PowerShell
