---
layout: post
title: "Skip reloading fixtures each time you run your Rails tests"
date: 2012-02-04 18:55
comments: true
categories:
---

Everyone wants their tests to run faster. I want my tests to run faster. That's why I've been using <a href="https://github.com/grosser/parallel">parallel</a> when I need to run my whole test suite (or at least a large part of it, e.g., unit tests). And <a href="https://github.com/myronmarston/vcr">VCR</a> makes it really easy to record HTTP interactions to play back later, which means your tests will run faster and will pass even when you're not online.

A lot of the time I just want to run a single test file (or single test method)—and this, for me, takes an annoyingly long time. If you have a lot of fixtures like I do, then you may want to read on. Let's start with the test benchmarks.

```
% time ruby test/unit/letter_test.rb
Loaded suite test/unit/letter_test
Started
..........
Finished in 7.705702 seconds.

10 tests, 10 assertions, 0 failures, 0 errors
10.80s user 1.90s system 15.393 total
```

Fifteen whole seconds to wait for a simple set of unit tests to run! and most of that time is spent just preparing to run the tests. My first step was to use <a href="https://github.com/sporkrb/spork.git">spork</a>, which speeds up testing by preloading the Rails env and forking when you want to run your tests.

```
% testdrb test/unit/letter_test.rb
Loaded suite letter_test.rb
Started
..........
Finished in 8.642509 seconds.

10 tests, 10 assertions, 0 failures, 0 errors
0.12s user 0.04s system 9.052 total
```

Quite an improvement! Spork shaved off about 6s from the total time to run the test just by preloading the Rails env.

But 9s is still along time to run a few simple tests. Digging through test.log I realized an absurd amount of time was being spent loading in fixtures.* The worst part is that there is really no need to reload the fixtures into the DB once they are in there as long as you are using transactional fixtures—I only need Rails to know which fixtures are present so I can easily access the AR objects I need by the names they have in the fixtures. This part doesn't take long at all. Most of the time loading in fixtures is spent putting them in the DB.

After some digging through ActiveSupport::TestCase and finding my way into ActiveRecord::Fixtures, I realized that if I could stub out the database adapter methods that are used to load in the fixtures then I could get the benefit of having Rails know which fixtures exist without actually spending the time to reload them into the database. Here's how I modified my test/test_helper.rb to achieve this using <a href="https://github.com/floehopper/mocha">mocha</a> (only relevant code shown):

``` ruby
ENV["RAILS_ENV"] = "test"
require File.expand_path('../../config/environment', __FILE__)
require 'rails/test_help'
require 'mocha'

unless (ENV["SKIP_FIXTURES"]||ENV["SF"]).nil?
  # Make sure to stub whatever adapter corresponds to your test db
  ActiveRecord::ConnectionAdapters::Mysql2Adapter.any_instance.stubs(:delete).returns(true)
  ActiveRecord::ConnectionAdapters::Mysql2Adapter.any_instance.stubs(:insert_fixture).returns(true)
end

class ActiveSupport::TestCase
  fixtures :all
  def setup
    ActiveRecord::ConnectionAdapters::Mysql2Adapter.any_instance.unstub(:delete)
    ActiveRecord::ConnectionAdapters::Mysql2Adapter.any_instance.unstub(:insert_fixture)
  end
  # other stuff...
end
```

Now when I run my test with the DB methods stubbed out:

```text
% time SF=true ruby test/unit/letter_test.rb
Loaded suite test/unit/letter_test
Started
..........
Finished in 2.664011 seconds.

10 tests, 10 assertions, 0 failures, 0 errors
7.96s user 1.70s system 10.562 total
```

And with spork (must start the spork server w/ SF=true):

```
% time testdrb test/unit/letter_test.rb
Loaded suite letter_test.rb
Started
..........
Finished in 3.351071 seconds.

10 tests, 10 assertions, 0 failures, 0 errors
0.14s user 0.01s system 3.574 total
```

So by using spork and skipping loading fixtures every time I was able to go
from 15s to ~3.5s. I can live with that. I can be productive with that.

\* This app was started a long, long time ago and migrating to factories would
  be way too painful.
