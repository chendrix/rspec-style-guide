# The RSpec Style Guide

This RSpec style guide recommends best practices so that real-world Ruby
programmers can write code that can be maintained by other real-world Ruby
programmers. A style guide that reflects real-world usage gets used, and a
style guide that holds to an ideal that has been rejected by the people it is
supposed to help risks not getting used at all &ndash; no matter how good it is.

The guide is separated into several sections of related rules. I've
tried to add the rationale behind the rules (if it's omitted I've
assumed that is pretty obvious).

The guide is still a work in progress - some rules are lacking
examples, some rules don't have examples that illustrate them clearly
enough. In due time these issues will be addressed - just keep them in
mind for now.

You can generate a PDF or an HTML copy of this guide using
[Transmuter](https://github.com/TechnoGate/transmuter).

## Table of Contents

* [RSpec](#rspec)
* [Mocking and Stubbing](#mocking-and-stubbing)
* [Plain Old Ruby Models](#plain-old-ruby-models)
* [ActiveRecord Models](#activerecord-models)
* [Mailers](#mailers)
* [Rake Tasks](#rake-tasks)
* [Miscellaneous](#miscellaneous)

## RSpec

* Try to use just one expectation per example, within reason.

    ```Ruby
    # bad
    describe ArticlesController do
      #...

      describe 'GET new' do
        it 'assigns new article and renders the new article template' do
          get :new
          assigns[:article].should be_a_new Article
          response.should render_template :new
        end
      end

      # ...
    end

    # good
    describe ArticlesController do
      #...

      describe 'GET new' do
        it 'assigns a new article' do
          get :new
          assigns[:article].should be_a_new Article
        end

        it 'renders the new article template' do
          get :new
          response.should render_template :new
        end
      end

    end
    ```

* Keep the full spec name (concatentation of the nested descriptions)
  grammatically correct.
  * Top level: use `describe` with a constant name: `describe User ...`
  * 2nd level: use `describe` with a method name: `describe "#awesome?"`
  * Inner blocks: use a `context` that starts with `when`: `context "when user is unsubscribed"`
  * Example describes the expectation: `it "is false"`.
  * Full spec name: "User#awesome? when user is unsubscribed is false"
 
* Do not use "should" in our example names.

  ```Ruby
  # good
  it "returns true"

  # bad
  it "should return true"

* Write expectations at a high level, removed from logic and implementation details.

  ```Ruby
  # bad
  it "calls more_results if i=0" do
    # ...
  end

  # good
  context "no results are returned by the initial search" do
    it "attempts to find more results" do
      # ...
    end
  end
  ```

* Use `describe` for concepts which don't in themselves vary (e.g. "callbacks, validations, <method_name>"). Typically these are nouns.

* Name `describe` blocks as follows:
  * use "description" for non-methods
  * use pound "#method" for instance methods
  * use dot ".method" for class methods

    ```Ruby
    class Article
      def summary
        #...
      end

      def self.latest
        #...
      end
    end

    # the spec...
    describe Article
      describe '#summary'
        #...
      end

      describe '.latest'
        #...
      end
    end
    ```

* Do not write iterators to generate tests.

  ```Ruby
  # bad
  [:new, :show, :index].each do |action|
    "it returns 200" do
      get action
      response.should be_ok
    end
  end
  ```

### Mocking and Stubbing

* Exclusively use mocking and stubbing in unit specs with POROs.
* Try to avoid mocking and stubbing in integration specs, favoring test parameters or
  attributes instead. When resorting to mocking and stubbing, only
  mock against a small, stable, obvious (or documented) API, so stubs
  are likely to represent reality after future refactoring.

* Other Valid reasons to use stubs/mocks:
  * Performance: To prevent running a slow, unrelated task.
  * Determinism: To ensure the test gives the same result each
    time. e.g. Time.now, Kernel#rand, external web services.
  * Vendoring: When relying on 3rd party code used as a "black box",
    which wasn't written with testability in mind.
  * Legacy: Stubbing old code that requires complex setup. (New code
    should not require complex setup!)
  * BDD: To remove the dependence on code that does not yet exist.

* Always use [Timecop](https://github.com/travisjeffery/timecop) instead of stubbing anything on Time or Date.

    ```Ruby
    # bad
    it "offsets the time 2 days into the future" do
      current_time = Time.now
      Time.stub(:now).and_return(current_time)
      subject.get_offset_time.should be_the_same_time_as (current_time + 2.days)
    end

    # good
    it "offsets the time 2 days into the future" do
      Timecop.freeze(Time.now) do
        subject.get_offset_time.should be_the_same_time_as 2.days.from_now
      end
    end
    ```

    NOTE: `#be_the_same_time_as` is a RSpec matcher we added to the platform, it
    is not normally available to RSpec.

### Plain Old Ruby Models

* Utilize mocking and stubbing to drive the design of your object's methods.
* Listen to the pain created during mocking and stubbing collaborators and dependencies to drive decoupling.

### ActiveRecord Models

* Do not mock the models in their own specs.
* Use Factory Girl to make real objects in these integration specs.
* It is acceptable to mock other models or child objects.
* Create the model for all examples in the spec to avoid duplication.
* Avoid unnecessary database calls.

    ```Ruby
    # use this:
    let(:article) { FactoryGirl.create(:article) }

    # ... instead of this:
    before(:each) { @article = FactoryGirl.create(:article) }
    ```

* Add an example ensuring that the fabricated model is valid.

    ```Ruby
    describe Article
      it 'is valid with valid attributes' do
        article.should be_valid
      end
    end
    ```

* When testing validations, use `have(x).errors_on` to specify the attibute
which should be validated. Using `be_valid` does not guarantee that the problem
 is in the intended attribute.

    ```Ruby
    # bad
    describe '#title'
      it 'is required' do
        article.title = nil
        article.should_not be_valid
      end
    end

    # preferred
    describe '#title'
      it 'is required' do
        article.title = nil
        article.should have(1).error_on(:title)
      end
    end
    ```

### Mailers

* The mailer spec should verify that:
  * the subject is correct
  * the receiver e-mail is correct
  * the e-mail is sent to the right e-mail address
  * testing the email body should be done in a view spec

     ```Ruby
     describe SubscriberMailer
       let(:subscriber) { FactoryGirl.build(:subscription, email: 'johndoe@test.com', name: 'John Doe') }

       describe 'successful registration email'
         subject { SubscriptionMailer.successful_registration_email(subscriber) }

         its(:subject) { should == 'Successful Registration!' }
         its(:from) { should == ['info@your_site.com'] }
         its(:to) { should == [subscriber.email] }
       end
     end
     ```

### Rake Tasks

* Rake tasks should be dumb (one-liners ideally), and call out to a model or
  library class which is tested like any other.

### Miscellaneous

* Avoid incidental state as much as possible.
    ```Ruby
    describe Article do
      let(:article) { FactoryGirl.create(:article) }
      let(:another_article) { FactoryGirl.create(:article) }

      describe "#publish" do
        # bad
        it "publishes the article" do
          article.publish
          
          # Creating another shared Article test object above would cause this
          # test to break
          Article.count.should == 2
        end

        # good
        it "publishes the article" do
          -> { article.publish }.should change(Article, :count).by(1)
        end
      end
    end
    ```
* Shared examples are helpful for tidying up repetitive expectations,
but should be written to be composable if possible to allow for situations where
many but not all of the shared expectations are required.

* Be careful not to focus on being "DRY" by moving repeated expectations
into a shared environment too early, as this can lead to brittle tests
that rely too much on one other.
