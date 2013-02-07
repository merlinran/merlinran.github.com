---
layout: post
title: RSpec中let的惰性求值
---
刚刚掉坑里了。上代码:

{% highlight ruby linenos %}
require 'spec_helper'  
  
describe User do  
  describe '#send_message' do  
    context 'when user is under their subscription limit' do  
      let(:john) { User.create! }  
      let(:sam) { User.create! }  
      let(:msg) { john.send_message(title: 'foo',  
                    text: 'bar',  
                    recipient: sam) }  
  
      it 'sends a message to another user' do  
        sam.received_messages.should == [msg]  
      end  
    end  
  end  
end
{% endhighlight %}

RSpec报错

{% highlight ruby linenos %}
Failures:  
  
  1) User#send_message when user is under their subscription limit sends a message to another user  
     Failure/Error: sam.received_messages.should == [msg]  
       expected: [#<Message id: 1, title: "foo", text: "bar", recipient_id: 1, created_at: "2012-07-03 10:58:25", updated_at: "2012-07-03 10:58:25">]  
            got: [] (using ==)  
       Diff:  
       @@ -1,2 +1,2 @@  
       -[#<Message id: 1, title: "foo", text: "bar", recipient_id: 1, created_at: "2012-07-03 10:58:25", updated_at: "2012-07-03 10:58:25">]  
       +[]  
     # ./spec/models/user_spec.rb:19:in `block (4 levels) in <top (required)>'  
{% endhighlight %}

找了半天没找出原因来。后来看了test.log里面的SQL记录，终于把问题找到了。下面的日志可以看到，对messages表的SELECT发生在INSERT之前。

{% highlight ruby %}
 (0.2ms)  RELEASE SAVEPOINT active_record_1  
Message Load (0.5ms)  SELECT "messages".* FROM "messages" WHERE "messages"."recipient_id" = 1  
 (0.1ms)  SAVEPOINT active_record_1  
SQL (0.6ms)  INSERT INTO "users" ("created_at", "login", "updated_at") VALUES (?, ?, ?)  [["created_at", Tue, 03 Jul 2012 10:45:27 UTC +00:00], ["login", nil], ["updated_at", Tue, 03 Jul 2012 10:45:27 UTC +00:00]]  
 (0.3ms)  RELEASE SAVEPOINT active_record_1  
 (0.1ms)  SAVEPOINT active_record_1  
SQL (1.2ms)  INSERT INTO "messages" ("created_at", "recipient_id", "text", "title", "updated_at") VALUES (?, ?, ?, ?, ?)  [["created_at", Tue, 03 Jul 2012 10:45:27 UTC +00:00], ["recipient_id", 1], ["text", "bar"], ["title", "foo"], ["updated_at", Tue, 03 Jul 2012 10:45:27 UTC +00:00]]  
 (0.2ms)  RELEASE SAVEPOINT active_record_1  
{% endhighlight %}

let(:msg)是定义一个叫msg的函数，而函数如果没调用，当然不执行了。所以，只有在调用前，触发INSERT操作。

而使用before(:each)就没有这样的问题，它是一定会执行的。
