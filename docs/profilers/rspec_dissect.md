# RSpecDissect 分析器

你知道在`before` hooks 上花费了多少时间吗？或者在诸如 `let` 之类的帮助方法上？通常，它们是整个测试套件中耗费时间最多的。

_RSpecDissect_ 提供了此类信息，还会向你展示最差的测试用例组。RSpecDissect 的主要目标是识别出那些慢的用例组，并使用 [`before_all`](../recipes/before_all.md) 或 [`let_it_be`](../recipes/let_it_be.md) 的配方来重构它们。

输出示例：

```sh
[TEST PROF INFO] RSpecDissect enabled

Total time: 25:14.870
Total `before(:each)` time: 14:36.482
Total `let` time: 19:20.259

Top 5 slowest suites (by `before(:each)` time):

Webhooks::DispatchTransition (./spec/services/webhooks/dispatch_transition_spec.rb:3) – 00:29.895 of 00:33.706 (327)
FunnelsController (./spec/controllers/funnels_controller_spec.rb:3) – 00:22.117 of 00:43.649 (133)
ApplicantsController (./spec/controllers/applicants_controller_spec.rb:3) – 00:21.220 of 00:41.407 (222)
BookedSlotsController (./spec/controllers/booked_slots_controller_spec.rb:3) – 00:15.729 of 00:27.893 (50)
Analytics::Wor...rsion::Summary (./spec/services/analytics/workflow_conversion/summary_spec.rb:3) – 00:15.383 of 00:15.914 (12)


Top 5 slowest suites (by `let` time):

FunnelsController (./spec/controllers/funnels_controller_spec.rb:3) – 00:38.532 of 00:43.649 (133)
 ↳ user – 3
 ↳ funnel – 2
ApplicantsController (./spec/controllers/applicants_controller_spec.rb:3) – 00:33.252 of 00:41.407 (222)
 ↳ user – 10
 ↳ funnel – 5
 ↳ applicant – 2
Webhooks::DispatchTransition (./spec/services/webhooks/dispatch_transition_spec.rb:3) – 00:30.320 of 00:33.706 (327)
 ↳ user – 30
BookedSlotsController (./spec/controllers/booked_slots_controller_spec.rb:3) – 00:25.710 of 00:27.893 e(50)
 ↳ user – 21
 ↳ stage – 14
AvailableSlotsController (./spec/controllers/available_slots_controller_spec.rb:3) – 00:18.481 of 00:23.366 (85)
 ↳ user – 15
 ↳ stage – 10
```

如你所见， `let` 分析器也追踪、提供了组内每个`let`声明被使用了多少次的信息（默认显示排在前三的）。

## 教学

RSpecDissect 只能跟 RSpec 一起使用（顾名思义）。

使用 `RD_PROF` 环境变量来激活 RSpecDissect：

```sh
RD_PROF=1 rspec ...
```

你也可以通过 `RD_PROF_TOP` 变量指定展示前几个最慢的组：

```sh
RD_PROF=1 RD_PROF_TOP=10 rspec ...
```

你还可以通过各自指定 `RD_PROF=let` 和 `RD_PROF=before`来仅追踪`let`或`before`的使用。

对于 `let` 分析器，你也可以通过 `RD_PROF_LET_TOP=10` 环境变量指定打印排位靠前的`let`声明的数量。

要禁用 `let` 统计，添加这个设置：

```ruby
TestProf::RSpecDissect.configure do |config|
  config.let_stats_enabled = false
end
```

## 与 RSpecStamp 一起使用

RSpecDissect 可以跟 [RSpec Stamp](../recipes/rspec_stamp.md) 一起使用，以自定义 tag 来自动标记 _慢_ 测试用例。例如：

```sh
RD_PROF=1 RD_PROF_STAMP="slow" rspec ...
```

运行上面命令后，最慢的测试用例组就以`:slow` tag 被标记了。
