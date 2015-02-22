---
layout: post
title: "Rails Test Details 1: Build Data"
date: 2015-2-5 11:47:06
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

今天的主题是构建测试数据。所以主要就是factory_girl和forgery了。

构建测试数据可以说是测试过程中最繁琐的一步了，很多时候为了测试一行代码你不得不准备N多的数据用来测试，尤其是当你测试一个controller方法的时候你可能需要构建超过10个的关联模型以及基本数据模型，所以熟悉构建数据会节省N多时间。

## 1. 简单数据

构建一个简单的数据非常容易，写两个demo：
{%highlight ruby%}
FactoryGirl.define do
  factory :user do
    email 'demo@test.com'
    password 'password'
    password_confirmation 'password'
  end
end

FactoryGirl.define do
  factory :profile do
    title Forgery::Name.academic_title
    first_name Forgery::Name.first_name
    last_name Forgery::Name.last_name
    gender Forgery::Personal.gender
    birthday Time.now - 35.years
  end
end
{%endhighlight%}
上面构建了user和profile两个独立的factory，当你想使用他们的时候可以`build(:user)`(＝User.new attrs), `create(:user)`(=User.create attrs)还有`attributes_for(:user)`(={attrs})。

Forgery的作用主要是生成一些假的信息，总比我们自己闭着眼睛想一些名字好。不过缺点是你不能直接预测你的数据内容用来测试。

比如当你想测试一个full_name方法时：
{%highlight ruby%}
def full_name
  [profile.first_name,profile.last_name].join(' ')
end

expect(profile.full_name).to eq([profile.first_name,profile.last_name].join(' '))
{%endhighlight%}
在这种场景下Forgery的意义并不是很大，仅仅是执行了两次同一个方法而已。

然后所以想在使用的时候替换某些属性，`create(:profile,first_name: 'Jerry', last_name: 'Tao')`，像这样，你就可以把你想替换的属性替换进去。

我觉得常用的场景是下面这种情况：

`expect(build(:profile,first_name:nil)).not_to be_valid`

## 2. 关联数据

关联数据也是很常见的一种情况，一般而言有三种情况来实现关联数据的模型(假设关系`user has_one profile`)：
{%highlight ruby%}
# 分别创建各自的model
profile = create(:profile)
user = create(:user,profile: profile)

# 使用association
FactoryGirl.define do
  factory :profile do
    title Forgery::Name.academic_title
    first_name Forgery::Name.first_name
    last_name Forgery::Name.last_name
    gender Forgery::Personal.gender
    birthday Time.now - 35.years
    association :user, factory: :user
  end
end

# 使用attributes_for 注意：仅适用于在user中有accepted_nested_attributes_for :profile的情况下
FactoryGirl.define do
  factory :user do
    email 'demo@test.com'
    password 'password'
    password_confirmation 'password'
    profile_attributes {
      attributes_for(:profile)
    }
  end
end
{%endhighlight%}
分别介绍一下三种方法，第一种方法是适用性最强的，基本可以在绝大多数情况下使用，但是有两个缺点。

一，在某些特定的验证下会失败，例如如果你在profile中 `validates :user_id, :presence`，在这种情况下你就没有办法先创建profile，并且如果同时你在user中验证profile的presence，那么你就陷入里一个死循环（这种验证是否合理还有待研究）

二，这种构建很复杂，尤其是多模型的时候。

第二种方法跟第一种方法实质是一样的，所以你也会面临上面问题一的尴尬场景，因为使用association的时候会优先create association。

然后说一下第三种方法，第三种方法我们假设业务是这样的，profile模型在创建user模型的同时被创建，并且参数是一次提交过来，即只通过accepted_nested_attributes_for这种情况来创建，所以第三种做法是最符合逻辑的。

你使用上面的哪种方法来创建model，可以参考你的业务逻辑，比如说商城系统，你需要一个注册用户，需要一个product，需要一个order，当你构建order的时候最好使用第一种方法或者第二种。

## 3. 构建list数据

当你想测试一个resources的index或者分页等情况的时候，很常见的是你需要一次性构建很多同一个模型的数据，如果这个模型并没有哪一列需要验证unique，你不需要修改任何东西可以直接使用：

`create_list(:profile,10,first_name:'10_times')`

这里第二个参数是你需要构建的数量，第二个参数后你依然可以自定义属性。

还有一种情况是这个模型包含unique列，例如user的email，那你就需要使用sequence：
{%highlight ruby%}
FactoryGirl.define do
  factory :user do
    sequence(:email) { |n| "demo_#{n}@test.com" }
    password 'password'
    password_confirmation 'password'
    profile_attributes {
      attributes_for(:profile)
    }
  end
end
{%endhighlight%}
上面并不会影响你单独使用create(:user)。

## 4. 根据不同情况构建数据

还是以user为例，让我们来添加一个模型，Setting，假设user有两种，一种是普通user，一种是admin，如果是admin的用户就需要关联一个setting模型：
{%highlight ruby%}
FactoryGirl.define do
  factory :user,class:'User' do
    sequence(:email) { |n| "demo_#{n}@test.com" }
    password 'password'
    password_confirmation 'password'
    profile_attributes {
      attributes_for(:profile)
    }

    factory :admin do
      sequence(:email) { |n| "admin_#{n}@test.com" } # 这里类似继承，user的属性都会有，并且可以在这里替换上面的属性或者增加一些别的属性
      after(:create) { |admin| admin.setting = create(:setting) } # 这里的回调类似active_record的回调，可以使用after(:build)或者before(:create)这种语法 这个也可以写到user里。
    end
  end
end
{%endhighlight%}
## 5. build_stubbed & create

如果你经常看测试，那么你很可能已经看到过build_stubbed这个方法了，这个方法的作用是构建一个完整内容的model

比如你使用`build_stubbed(:profile)`(不包含association的profile)，那么你的这个profile模型会包含id，和user_id（都是一些假的值，并且这个模型并没有被存储进数据库）。

所以在合理使用build_stubbed的时候会加快测试的速度，并且减少了构建关联模型的时间。

比如说你想测试一个order，这个order同时关联里一个product，可是如果你想create一个product还需要创建包含卖家，卖家信息等一系列关联model的时候，你可以像下面这样来构建order：
{%highlight ruby%}
product = build_stubbed(:product)
order = build(:order,product: product)
expect(order).to be_valid
{%endhighlight%}
这里的核心是千万不要在你想测试的模型上使用build_stubbed。

比如上面的测试，你不应该`build_stubbed(:order)`，因为`build_stubbed`会自动帮你补全缺失的属性，所以如果你使用`build_stubbed(:order)`构建出来的order测试valid，基本都会通过测试。
