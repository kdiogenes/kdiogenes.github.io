---
title: Exploring Campfire Tests
date: 2024-10-28
category: [rails, tests]
tags: [rails, tests, minitest]
---

I was always curious to see real [37signals](https://37signals.com/) code. When they released their first [ONCE](https://once.com/) product, [Campfire](https://once.com/campfire), I was tempted to buy it but I was not willing to spend US$ 299,00, the release price, and now it's US$ 399,00. For me, it was too much money.

Some time ago, they released special prices for some countries, and I bought it for R$ 499,00, approximately US$ 90,00 today. I still think that it's too much money just to see code written by someone else, but I was curious to see how they write code, especially their test code suite.

I've already read some test code from Rails, but a framework is not an end-user product, is the focus different in this case? Furthermore, I always struggled to write tests to the project I'm working on, so I thought that this can be a good opportunity to learn more about this practice that I was never able to fully put in practice.

## Campfire Codebase Overview

Before digging into the test code, I would like to first run it and got some statistics about the codebase. To be able to run the tests I decided to create a [Dev Container](https://code.visualstudio.com/docs/devcontainers/containers).

### Dev Container Configuration

I just copied the content of a .devcontainer folder that is generated from a new Rails project when using the `--devcontainer` flag.

```yaml
# .devcontainer/compose.yaml
name: "campfire"

services:
  rails-app:
    build:
      context: ..
      dockerfile: .devcontainer/Dockerfile

    volumes:
    - ../..:/workspaces:cached

    # Overrides default command so things don't shut down after the process ends.
    command: sleep infinity

    # Uncomment the next line to use a non-root user for all processes.
    # user: vscode

    # Use "forwardPorts" in **devcontainer.json** to forward an app port locally.
    # (Adding the "ports" property to this file will not forward from a Codespace.)
    depends_on:
    - selenium

  selenium:
    image: selenium/standalone-chromium
    restart: unless-stopped
```

```json-doc
// .devcontainer/devcontainer.json
// For format details, see https://aka.ms/devcontainer.json. For config options, see the
// README at: https://github.com/devcontainers/templates/tree/main/src/ruby
{
  "name": "campfire",
  "dockerComposeFile": "compose.yaml",
  "service": "rails-app",
  "workspaceFolder": "/workspaces/${localWorkspaceFolderBasename}",

  // Features to add to the dev container. More info: https://containers.dev/features.
  "features": {
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/rails/devcontainer/features/activestorage": {},
    "ghcr.io/rails/devcontainer/features/sqlite3": {},
    // Added node to be able to install playwright
    "ghcr.io/devcontainers/features/node:1": {}
  },

  "containerEnv": {
    "CAPYBARA_SERVER_PORT": "45678",
    "SELENIUM_HOST": "selenium"
  },

  // Use 'forwardPorts' to make a list of ports inside the container available locally.
  "forwardPorts": [3000],

  // Configure tool-specific properties.
  // "customizations": {},

  // Uncomment to connect as root instead. More info: https://aka.ms/dev-containers-non-root.
  // "remoteUser": "root",

  // Use 'postCreateCommand' to run commands after the container is created.
  "postCreateCommand": "bin/setup --skip-server"
}
```

```dockerfile
# .devcontainer/Dockerfile
# Make sure RUBY_VERSION matches the Ruby version in .ruby-version
ARG RUBY_VERSION=3.3.1
FROM ghcr.io/rails/devcontainer/images/ruby:$RUBY_VERSION
```

### Tests Execution

After this, I was able to reopen the project in a Dev Container and run the tests. In the first run I got an error due the lack of two files: `test/fixtures/files/alpha-century.mov` and `test/fixtures/files/moon.jpg`. I just download a moon image from the internet and downloaded a mov file from https://file-examples.com/index.php/sample-video-files/sample-mov-files-download/. An important detail is that the image must have a square aspect ratio.

After this, the test suite run without errors:

```bash
vscode ➜ /workspaces/campfire $ bin/rails test
Run options: --seed 21417

# Running:

...............................................................................................................................................................................................................................

Finished in 26.613048s, 8.3793 runs/s, 23.7853 assertions/s.
223 runs, 633 assertions, 0 failures, 0 errors, 0 skips
```

On the `test_helper.rb` file the parallelize method call is commented with a fix note:

```ruby
# FIXME: sqlite3 isn't correctly creating the additional databases per core
# parallelize(workers: :number_of_processors)
```

I tried to uncomment and run the tests anyway, and for my surprise, they worked:

```bash
vscode ➜ /workspaces/campfire $ bin/rails test
Running 223 tests in parallel using 16 processes
Run options: --seed 46877

# Running:

...............................................................................................................................................................................................................................

Finished in 4.631673s, 48.1468 runs/s, 136.6677 assertions/s.
223 runs, 633 assertions, 0 failures, 0 errors, 0 skips
```

A pretty good time. If you try to run the tests again, you will start to get some errors. You can delete the tests databases with the following command: `rm storage/db/test.sqlite3*`. After this, you can run the tests again, and they will pass.

I also tried to run the system tests and after some hours trying to make it work, I was almost giving up when I stumbled on this post: https://justin.searls.co/posts/running-rails-system-tests-with-playwright-instead-of-selenium/. Knowing the work of Justin Searls in the Ruby community, I though this was worth a try.

The process to use it was quite simple:

- Added the feature `"ghcr.io/devcontainers/features/node:1": {}` to `.devcontainer/devcontainer.json`
- Rebuild the Dev Container
- Replace `gem "selenium-webdriver"` with `gem "capybara-playwright-driver"`
- Run `bundle` to update the gems
- Install playwright:

```bash
export PLAYWRIGHT_CLI_VERSION=$(bundle exec ruby -e 'require "playwright"; puts Playwright::COMPATIBLE_PLAYWRIGHT_VERSION.strip')
npm install -g "playwright@$PLAYWRIGHT_CLI_VERSION"
npx playwright install --with-deps
```

After this I just changed the minitest system tests to be `driven_by :playwright, using: :headless_chrome, screen_size: [ 1400, 1400 ]` and the tests executed successfully:

```bash
vscode ➜ /workspaces/campfire $ bin/rails test:system
Run options: --seed 50942

# Running:

Capybara starting Puma...
* Version 6.4.2 , codename: The Eagle of Durango
* Min threads: 0, max threads: 4
* Listening on http://127.0.0.1:44127
........

Finished in 141.722643s, 0.0564 runs/s, 0.4234 assertions/s.
8 runs, 60 assertions, 0 failures, 0 errors, 0 skips
```

To get the statistics about the test codebase, I just used the stats command from Rails:

```bash
vscode ➜ /workspaces/campfire $ bin/rails stats
+----------------------+--------+--------+---------+---------+-----+-------+
| Name                 |  Lines |    LOC | Classes | Methods | M/C | LOC/M |
+----------------------+--------+--------+---------+---------+-----+-------+
| Controllers          |   1102 |    883 |      32 |     172 |   5 |     3 |
| Helpers              |    662 |    574 |       4 |      83 |  20 |     4 |
| Jobs                 |     17 |     12 |       3 |       2 |   0 |     4 |
| Models               |   1205 |    960 |      27 |     156 |   5 |     4 |
| Channels             |     91 |     79 |       8 |      14 |   1 |     3 |
| Views                |   2199 |   1932 |       0 |       3 |   0 |   642 |
| Stylesheets          |   3312 |   2705 |       0 |       0 |   0 |     0 |
| JavaScript           |   3826 |   3052 |       0 |      46 |   0 |    64 |
| Libraries            |    246 |    205 |       8 |      29 |   3 |     5 |
| Controller tests     |   1194 |    929 |      29 |     123 |   4 |     5 |
| Helper tests         |     93 |     72 |       1 |      12 |  12 |     4 |
| Model tests          |    948 |    753 |      20 |      96 |   4 |     5 |
| Channel tests        |     72 |     52 |       2 |       9 |   4 |     3 |
| System tests         |    196 |    157 |       3 |      11 |   3 |    12 |
+----------------------+--------+--------+---------+---------+-----+-------+
| Total                |  15163 |  12365 |     137 |     756 |   5 |    14 |
+----------------------+--------+--------+---------+---------+-----+-------+
  Code LOC: 10402     Test LOC: 1963     Code to Test Ratio: 1:0.2
```

I also installed the [simplecov](https://github.com/simplecov-ruby/simplecov) gem. For the tests the coverage is of 95.94% and for the system test it's of 78.3%. Pretty good. I save the report for the [tests](/assets/pages/2024/campfire_coverage/tests/#_AllFiles) and the [system tests](/assets/pages/2024/campfire_coverage/system_tests/#_AllFiles) if you want to take a look.

The codebase is pretty small, with around 10k lines of code and 2k lines of test code. It's interesting to note that views, stylesheets and JavaScript are more than 75% of the codebase, and they have almost no tests. The majority of tests are for controllers and models, having almost the same number of lines of the actual code.

Although there isn't specific tests for views, it's important to note that the controllers tests are [functional tests](https://guides.rubyonrails.org/testing.html#functional-tests-for-your-controllers), exercising routes, controllers and views. So, there is a bit of coverage for views, but as we will see in more details later, the focus on views is minimal.

## Exploring the Test Suite

The first tests I started exploring were the system tests. I thought that this could give a good understand of the overall functionality of the system, but after opening the first test file, `boosting_messages_test.rb`, the first thing I thought was: "what the hell boosting a message means?". After reading a bit this test file, the model and mainly by running the application I understood that boosting a message is reacting to a message with emojis and texts.

Although this initial confusion, the test is very well written, the setup is simple and the scenarios are very easy to read. Here is a list of the test descriptions on this file:

- boosting a message
- deleting a boost
- message update preserves the input state
- boost by another user preserves the input state

The process of boosting messages is complex, not the process itself, but the real-time communication that must be done, so this test focus on some edge cases on how the DOM reacts to it.

The next file on the system test folder is `sending_messages_test.rb`. The tests here are also very simple, and here are the descriptions of the test cases on this file:

- sending messages between two users
- editing messages
- deleting messages

I think that the main motivation for these tests were also due the real-time complexity. Like the boosting message tests, these also involves multiple users and verifications that the operations reflects on the various users screens.

The last file on this folder is `unread_rooms_test.rb`. This file contains only one test case named `sending messages between two users` that asserts about the read/unread status of chatting rooms.

Although that isn't many tests, my perception is that they are exercising the most complex part of the system and also the most used part of the system. So, considering costs and benefits it appears to be OK (only these tests take 2 minutes to run). I'm a bit paranoid about important workflows, and one that I think I would have tested is the invitation feature to join chat rooms, but given the recent DHH writing on [System tests have failed](https://world.hey.com/dhh/system-tests-have-failed-d90af718), it is understandable that there is no more system testing.

It's also important to note that only these tests covers more than [75% of the application logic](/assets/pages/2024/campfire_coverage/system_tests/#_AllFiles). The other tests cover more than 95% of the application logic, so for sure the invitation logic is covered, but it's almost backend logic. Anyway, many things happen while a system is being implemented, and I don't want to be overly speculative because in the projects I participated in, there were always parts of the system that we knew could be better, but we decided to ignore to focus on things we considered more importantly.

### Controller Tests

Just like the system tests, the controller tests are also very well written, with simple setup and very easy follow, with each test scenario having only a couple of lines (approximately 3 to 4 lines per test).

I got really surprised by the simplicity of these tests. They basically exercise the controller and verifies that the response code is the expected one or that the redirection is to the expected URL.

Recently I read the principles' section of [Professional Rails Testing](https://www.amazon.com/Professional-Rails-Testing-Tools-Principles/dp/B0DJRLK93M) and in the book, Jason Swett, states that if your code is a mess, your tests will also be a mess. Probably my surprise was due the simplicity of the design when compared with the main codebase I have been working in the last years: a total mess!

Almost all the controller tests on Campfire exercise all the actions implemented on the corresponding controllers, with very rare exceptions. The tests also exercise some edge cases for actions, in general non-admin restrictions, a policy that permeates various controllers, and some edge cases for specific controllers.

For example, the `accounts/logos_controller_test.rb` implements the following cases for the show action:

- show stock
- show stock small size
- show custom
- show custom small size

Another controller test with edge cases is for the index action of the controller is `autocompletable/users_controller_test.rb`:

- search returns matching users
- search results escape HTML in names
- room search returns matching users
- room search is scoped by membership

I don't think there is much more to discuss regarding these tests. One thing I noticed is that the controller test descriptions don’t provide much explanation of the *why* behind the application’s behavior. Perhaps this is due to my focus on BDD and the outside-in strategy for implementing tests. This approach encourages writing test suites as specifications, making the *why* more explicit in the test descriptions.

I also don't want to discuss whether this is a better approach, as that would be speculative since I haven’t had much opportunity to explore this BDD style of writing tests. I’m starting a greenfield project where I plan to explore this concept more rigorously and hopefully write about the experience. Currently, I think writing tests as specifications is more challenging, but I believe it’s worth a try because it can help new programmers understand the codebase more easily and quickly.

One of the things that I think can help to pursue documenting more the *whys* of the application with tests as specifications can be writing more integration tests. The actual controller tests of Campfire are already integration tests, so for me, it will be a quest to find the right words and write tests that exercise the application according to the specifications. Almost system tests, but without all the heavy load.

### Model Tests

One of the first things I looked in these tests was for [tautological tests](https://www.codewithjason.com/examples-pointless-rspec-tests/). I was really not expecting to find them, and they really aren't present on this codebase. All the tests exercise the behaviors that the model must exhibit.

I remember a long time ago, when I first started reading about testing Rails applications, I discovered the [shoulda-matchers](https://github.com/thoughtbot/shoulda-matchers) gem. Back then, the sentiment was, 'Cool, it’s so much easier to test relationships, validations, etc.' Unfortunately, there is almost no behavior testing with these matchers—only a different way to restate what you’ve already written in your model. You might think, 'If the programmer wrote it twice, it must be important!' But I don’t believe that’s the case. These tests don’t help in understanding more about the application's behavior and can be a hassle when refactoring.

Now let's come back to the Campfire test suite. Once again I was impressed by the simplicity of the tests' setup. In this case, most of them doesn't have a before block, they only load a fixture, apply some actions and assert the results.

A test file that I have some difficult to understand was the `action_text_attachment_test.rb`. This doesn't test a specific model of the application, but some modifications that the application does to [ActionText](https://github.com/rails/rails/tree/main/actiontext). After realizing this, it was clear that mentions of users on the chat, that is handled by ActionText::Attachable, continues to work only for the `User` model when the `SECRET_KEY_BASE` value gets changed.

Other tests that catch my attention were the ones present on the file `membership_test.rb`. The model `Membership` is used to hold the information of the rooms a user can access and the user involvement (permissions) with the rooms. What I found curious is that the tests only exercises the behavior related with a secondary concern of the membership model, that is related with connections. The test descriptions are simple, so you have to infer the *whys* by reading the test code. Although I think that for the majority of the tests it's not a difficult task, but this case really catch my attention:

```ruby
test "connecting" do
  @membership.connected
  assert @membership.connected?
  assert_equal 1, @membership.connections

  @membership.connected
  assert_equal 2, @membership.connections
end
```

The test description isn't pretty in my opinion, the `connected` method changes the `connected?` response and each time it's called the `connections` counter is incremented. This is one of the cases where you have to understand better how the application is implemented to understand how this behavior is used and why the test is important.

Other model tests that catch my attention were the ones in the `test/model/opengraph` dir. These are some models implemented to deal with the [Open Graph Protocol](https://ogp.me/). These are the most extensive model tests, although they aren't the most important part of the core application, but there are many edge cases that must be addressed for what they want. Another thing is that the these tests needs mocks and stubs, something that largely contributes to make the test files bigger. Anyway, these concepts are very neatly applied, and the tests are very easy to follow.

# Final Considerations About the Test Suite

Overall the tests are coherent with many characteristics considered valuable by the testing community:

- Simple test setup
- Rapid execution
- [Testing pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)
- Well defined
- Straightforward

Maybe pointing them as straightforward is a bit misleading according to my previous comments. Although I think that they could be written more like a specification, the test code and the application are simple enough to understand what is going on for anyone that already developed some Rails applications.

One aspect that can influence this kind of detail is how the tests were written. We can't know if they were written before or after the code, the Campfire code is not delivered as a git repository, so we can't track the code evolution.

This exploration also showed me that there isn't a secret for testing. Probably the great secret is not how to test, but how to develop testable systems. When I started to write tests this was the first benefit I perceived. You take the idea in your head, and you start to think about it more thoroughly, you start to think more about how you can design it and all the interactions of the small details. For me, this always pushed the code toward simpler implementations and solutions, and consequently simpler designs.

Although I still believe that a test suite can be used to document the *whys* of your system, I also understand that it's very difficult, I even consider it impractical, to capture and document all your decisions through a test suite.

I think that this paragraph has a big essence of the Campfire test suite:

> *Which gets to the heart of why we automate testing. We do it for the quick feedback loop on changes, we do it to catch regressions, but most of all, we do it to become confident that the system works.*
>
> *-- DHH ([System tests have failed](https://world.hey.com/dhh/system-tests-have-failed-d90af718))*

This test suite exercises much of the application logic, so you can also have confidence in upgrading the framework, do refactors and implement new features.

Finally, there are a very small quantity of tests for channels and helpers. There is also a folder called `test/performance` that appears to test the performance of the application handling websocket connections and messages, but I didn't find any documentation or task to run them.

# Other Considerations

I perceived some cool things on the code during this journey that I think is cool to share.

Some places use a really neat way to nest blocks:

```ruby
assert_turbo_stream_broadcasts [ users(:david), :rooms ], count: 1 do
assert_turbo_stream_broadcasts [ users(:kevin), :rooms ], count: 1 do
assert_turbo_stream_broadcasts [ users(:jason), :rooms ], count: 1 do
  post rooms_closeds_url, params: { room: { name: "My New Room" }, user_ids: [ users(:david).id, users(:kevin).id, users(:jason).id ] }
end
end
end
```

I'm sure, it's one of the scenarios that DHH like to have the freedom to write the code the way he likes, without a code formatter changing it or without rubocop saying it's not the right way to write nested blocks. I really liked this formatting, for me, it's much easier to read. Compare with the nested block with indentation:

```ruby
assert_turbo_stream_broadcasts [ users(:david), :rooms ], count: 1 do
  assert_turbo_stream_broadcasts [ users(:kevin), :rooms ], count: 1 do
    assert_turbo_stream_broadcasts [ users(:jason), :rooms ], count: 1 do
      post rooms_closeds_url, params: { room: { name: "My New Room" }, user_ids: [ users(:david).id, users(:kevin).id, users(:jason).id ] }
    end
  end
end
```

Another thing I had never paid attention to, is that fixtures are a proxy to ActiveRecord (I always used FactoryBot). Take a look at this test example:

```ruby
test "update" do
  assert users(:david).administrator?

  put account_user_url(users(:david)), params: { user: { role: "administrator" } }

  assert_redirected_to edit_account_url
  assert users(:david).reload.administrator?
end
```

The last assertion uses `users(:david)` to reference the ActiveRecord object and reload the object from the database to verify that the administrator flag changed to true.

There is also a controller called `UnfurlLinksController` that is responsible to extract opengraph metadata from URLs. I'm Brazilian and the word `Unfurl` wasn't in my vocabulary. Right away, I though this was some play with the word URL, but after searching for the meaning I found it on the dictionary and understand what it means. This is one more example of 37signals showing to us how we can have joy with code by selecting a vocabulary that is meaningful and will probably make the concept stick harder on the programmer mind.

Another example of how we can have joy is this test example:

```ruby
test "all emoji" do
  assert Message.new(body: "😄🤘").plain_text_body.all_emoji?
  assert_not Message.new(body: "Haha! 😄🤘").plain_text_body.all_emoji?
  assert_not Message.new(body: "🔥\nmultiple lines\n💯").plain_text_body.all_emoji?
  assert_not Message.new(body: "🔥 💯").plain_text_body.all_emoji?
end
```

This remembers me about this Rails commit: https://github.com/rails/rails/commit/22af62cf486721ee2e45bb720c42ac2f4121faf4 and some people arguing how "professional" it was and others issues. What was DDH response? Add a `forty_two` method to `Array`: http://github.com/rails/rails/commit/e50530ca3ab5db53ebc74314c54b62b91b932389. Awesome!

Happy hacking!