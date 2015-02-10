---
layout: post
title: "Rails Test Details 2: Model Test"
date: 2015-2-6 11:47:06
categories: Rails
image: /assets/images/post.jpg
---

最近一直在跟测试打交道，遇见了N多爬不完的坑，所以在这里总结一下一些常见的坑。

测试栈：

- `rails 4.2`
- `rspec-rails 3.1`
- `factory_girl_rails`
- `database_cleaner`
- `simplecov`
- `forgery`

数据库： `Postgresql 9.6`

操作系统： `Mac OS 10.9`

今天的主题是model。所以在开始之前我们需要先回顾一下rails的model。

首先我们先考虑一下model里面都有什么：

- attributes
- assocations
- validations
- callbacks
- methods
- scopes
- concerns

上面的内容中，attributes和单纯的associations我觉得没有测试的必要，因为如果这两部分出错，在migrates和测试validations时刻就可以检测出来，所以这两部分跳过。

# 1. Validations

我觉得validations部分需要测试两个点，首先你要确保你想要validate的属性都加上了validations。

{%highlight ruby%}
describe User do
  let(:user) { build(:user) }
  subject { user }

  describe 'validations' do
    it { should validate_presence_of(:first_name) }
    it { should validate_presence_of(:last_name) }
  end
end
{%endhighlight%}

其次你要确保构建一个valid的factory，这一步也是其他测试包括controller测试的基础。

{%highlight ruby%}
describe User do
  let(:user) { build(:user) }
  subject { user }

  describe 'validations' do
    ...
    it { is_expected.to be_valid }
  end
end
{%endhighlight%}

# 2. Callbacks & Methods & Scopes

其实callbacks可以跟方法归为一类，但是会有一个skip的情况。

先看callback：

{%highlight ruby%}
class User < ActiveRecord::Base
  after_create :send_confirm_email

  private
  def send_confirm_email
    ...
  end
end
{%endhighlight%}

比如你不想在valid的时候就验证这个callback，最好的办法是修改一下你的factory：

{%highlight ruby%}
FactoryGirl.define do
  factory :user,class:'User' do
    sequence(:email) { |n| "demo_#{n}@test.com" }
    password 'password'
    password_confirmation 'password'
    after(:build) { |user| user.class.skip_callback(:create,:after,:send_confirm_email) } # 跳过callback
    factory :send_email_user do
      after(:create) { |user| user.send(:send_confirm_email)}
    end
  end
end
{%endhighlight%}

在上面的factory中，使用`create(:user)`不会发送邮件，`create(:send_email_user)` 会发送。

至于方法的测试其实真的很容易，还是用name来举例吧。

{%highlight ruby%}
class User < ActiveRecord::Base

  scope :admins, lambda{ where(role: :admin) }
  scope :normal_users, lambda { where(role: :normal) }
  ...
  def name
    [self.first_name, self.last_name].join(' ')
  end
end

#methods
describe '#methods' do
  it 'should give the full name' do
    expect(user).to eq([user.first_name, user.last_name].join(' '))
  end
end
#scopes
describe '#methods' do
  before(:each){
    #如何构建不同的user请参考上一篇文章
    create_list(:user,10)
    create_list(:admin,2)
  }
  it 'should count 10 normal user' do
    expect(User.normal_users).to eq(10)
  end
  it 'should count 2 normal user' do
    expect(User.admins).to eq(2)
  end
end
{%endhighlight%}

# 3. Concerns

之所以把concerns单独拿出来说，是因为concerns经常是复用的，包括很多第三方的gem的内容。

rspec在这一点上也是可以复用的，并不仅仅是concerns，任何重复的内容都可以拿出来单独的测试。(这里我仅仅是把name拿出来单独测试一下)

{%highlight ruby%}
#spec/models/user_spec.rb
describe User
  it_behaves_like 'model with name'
end

#spec/shared/shared_examples_model_with_name.rb
shared_examples_for 'model with name' do

  let(:factory_name) { described_class.name.underscore.to_sym }
  let(:resource) { create(factory_name) }

  describe '#methods' do
    it 'should give the full name' do
      expect(resource).to eq([resource.first_name, user.last_name].join(' '))
    end
  end
end
{%endhighlight%}

Model的测试更多的时候不仅仅是为了保证你的方法是正确的，而且是重构代码的重要保证手段。 方法的测试并没有太多的大的难题，但是还是有很多细节的，还有具体都应该测试什么情况，这些情况去找一本软件测试的书看看吧。
