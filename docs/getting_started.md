# 快速开始

## 安装

把`test-prof` gem 添加到你的应用：

```ruby
group :test do
  gem "test-prof", "~> 1.0"
end
```

完成！现在就可以使用 TestProf 的[分析器](/#profilers)了。

## 配置

TestProf 的全局配置被大多数分析器使用：

```ruby
TestProf.configure do |config|
  # the directory to put artifacts (reports) in ('tmp/test_prof' by default)
  config.output_dir = "tmp/test_prof"

  # use unique filenames for reports (by simply appending current timestamp)
  config.timestamps = true

  # color output
  config.color = true

  # where to write logs (defaults)
  config.output = $stdout

  # alternatively, you can specify a custom logger instance
  config.logger = MyLogger.new
end
```

你也可以通过`TEST_PROF_REPORT`环境变量动态添加 artifacts/reports 后缀。如果你没使用时间戳并想要用不同设置生成多个报告且比较它们时，这将很有用。

例如，让我们使用[`stackprof`](./profilers/stack_prof.md)来比较下测试用和不用`bootsnap`的加载时间:

```sh
# Generate first report using `-with-bootsnap` suffix
$ TEST_STACK_PROF=boot TEST_PROF_REPORT=with-bootsnap bundle exec rake
$ #=> StackProf report generated: tmp/test_prof/stack-prof-report-wall-raw-boot-with-bootsnap.dump

# Assume that you disabled bootsnap and want to generate a new report
$ TEST_STACK_PROF=boot TEST_PROF_REPORT=no-bootsnap bundle exec rake
$ #=> StackProf report generated: tmp/test_prof/stack-prof-report-wall-raw-boot-no-bootsnap.dump
```

现在你就有了两个带清晰命名的 stackprof 报告！