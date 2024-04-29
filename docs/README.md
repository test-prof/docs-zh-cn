[![Gem Version](https://badge.fury.io/rb/test-prof.svg)](https://rubygems.org/gems/test-prof) [![Build](https://github.com/test-prof/test-prof/workflows/Build/badge.svg)](https://github.com/test-prof/test-prof/actions)
[![JRuby Build](https://github.com/test-prof/test-prof/workflows/JRuby%20Build/badge.svg)](https://github.com/test-prof/test-prof/actions)

# TestProf

> Ruby 测试的分析和优化工具箱。

<img align="right" height="150" width="129"
     title="TestProf logo" class="home-logo" src="/assets/images/logo.svg">

TestProf 是一个分析你的测试套件性能的不同工具集。

为什么测试套件性能如此重要？ 首先，测试是开发者反馈环的一部分（参看 [@searls](https://github.com/searls) [talk](https://vimeo.com/145917204)）。其次，它是开发周期的一部分。

简单来说，慢测试浪费你的时间，让你效率低下。

TestProf 工具箱旨在帮你识别测试套件的瓶颈。它包含：

- 为常规 Ruby 分析器（[`ruby-prof`](https://github.com/ruby-prof/ruby-prof), [`stackprof`](https://github.com/tmm1/stackprof)）提供了即插即用的整合。

- 对 Factories 的使用采用了分析器。

- ActiveSupport 所支持的分析器。

- 提供了 RSpec 和 minitest 的 [帮助方法](#recipes) 以编写更快的测试。

- RuboCop 的支持。

- 更多……

📑 [Documentation](https://test-prof.evilmartians.io)

<p align="center">
  <a href="http://bit.ly/test-prof-map-v1">
    <img src="/assets/images/coggle.png" alt="TestProf map" width="738">
  </a>
</p>

<p align="center">
  <a href="https://evilmartians.com/?utm_source=test-prof">
    <img src="https://evilmartians.com/badges/sponsored-by-evil-martians.svg"
         alt="Sponsored by Evil Martians" width="236" height="54">
  </a>
</p>
## 使用 TestProf 的用户

- [Discourse](https://github.com/discourse/discourse) 减少了 [他们测试套件约 27% 的耗时](https://twitter.com/samsaffron/status/1125602558024699904)
- [Gitlab](https://gitlab.com/gitlab-org/gitlab-ce) 减少了 [他们 API 测试 39% 的耗时](https://gitlab.com/gitlab-org/gitlab-ce/merge_requests/14370)
- [CodeTriage](https://github.com/codetriage/codetriage)
- [Dev.to](https://github.com/thepracticaldev/dev.to)
- [Open Project](https://github.com/opf/openproject)
- [其他更多……](https://github.com/test-prof/test-prof/issues/73)

## 资源列表

- [TestProf: Ruby慢测试的“良医圣手”](https://xfyuan.github.io/2020/07/testprof-doctor-for-slow-ruby-tests/)

- [TestProf II: Ruby测试的“工厂疗法”](https://xfyuan.github.io/2020/07/testprof-factory-therapy-for-ruby-tests/)

- Paris.rb, 2018, “慢测试的99个问题” 演讲 [[视频](https://www.youtube.com/watch?v=eDMZS_fkRtk), [slides](https://speakerdeck.com/palkan/paris-dot-rb-2018-99-problems-of-slow-tests)]

- BalkanRuby, 2018, “带你的慢测试去看‘良医’” 演讲 [[视频](https://www.youtube.com/watch?v=rOcrme82vC8)], [slides](https://speakerdeck.com/palkan/balkanruby-2018-take-your-slow-tests-to-the-doctor)]

- RailsClub, Moscow, 2017, “更快的测试” 演讲 [[视频](https://www.youtube.com/watch?v=8S7oHjEiVzs) (俄语), [slides](https://speakerdeck.com/palkan/railsclub-moscow-2017-faster-tests)]

- RubyConfBy, 2017, “测试跑跑跑” 演讲 [[视频](https://www.youtube.com/watch?v=q52n4p0wkIs), [slides](https://speakerdeck.com/palkan/rubyconfby-minsk-2017-run-test-run)]

- [提升你测试套件速度的技巧](https://medium.com/appaloosa-store-engineering/tips-to-improve-speed-of-your-test-suite-8418b485205c) 演讲者：[Benoit Tigeot](https://github.com/benoittgt)

## 安装

把 `test-prof` gem 添加到你的应用：

```ruby
group :test do
  gem "test-prof"
end
```

就行了！

所支持的 Ruby 版本：

- Ruby (MRI) >= 2.5.0（**注意：** 对于 Ruby 2.2 请使用 TestProf < 0.7.0，Ruby 2.3 请使用 TestProf ~> 0.7.0，Ruby 2.4 请使用 TestProf <0.12.0）

- JRuby >= 9.1.0.0 （**注意** refinements-dependent 特性可能需要 9.2.7+）

所支持的 RSpec 版本（仅 RSpec 特性）: >= 3.5.0（对于旧版本 RSpec 请使用 TestProf < 0.8.0）。

## 分析器

- [RubyProf 的整合](./profilers/ruby_prof.md)

- [StackProf 的整合](./profilers/stack_prof.md)

- [Event 分析器](./profilers/event_prof.md)（比如，ActiveSupport 的 notifications）

- [Tag 分析器](./profilers/tag_prof.md)

- [Factory 医生](./profilers/factory_doctor.md)

- [Factory 分析器](./profilers/factory_prof.md)

- [RSpecDissect 分析器](./profilers/rspec_dissect.md)

## 配方

我们也期望分享一些小的代码技巧，可以帮助你改进测试套件的性能和效率：

- [`before_all` Hook](./recipes/before_all.md)

- [`let_it_be` 帮助方法](./recipes/let_it_be.md)

- [AnyFixture](./recipes/any_fixture.md)

- [FactoryDefault](./recipes/factory_default.md)

- [FactoryAllStub](./recipes/factory_all_stub.md)

- [RSpec Stamp](./recipes/rspec_stamp.md)

- [Tests Sampling](./recipes/tests_sampling.md)

- [Active Record Shared Connection](./recipes/active_record_shared_connection.md)

- [Rails 日志记录](./recipes/logging.md)

## 其他工具

- [RuboCop cops](./misc/rubocop.md)

## 下一步

有好的想法？[提交](https://github.com/test-prof/test-prof/discussions)一个功能需求吧!

已经在使用 TestProf 了？[分享你的故事吧](https://github.com/test-prof/test-prof/discussions/73)

## 许可协议

根据 [MIT License](http://opensource.org/licenses/MIT) 条款，本 gem 可作为开源使用。

