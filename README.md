# The Zen of Test Suites

Serious testing for serious software (work in progress)

## Introduction

**Note**: This document is about testing applications. They require a
different, more disciplined approach than testing libraries. I describe common
mis-features experienced in large application test suites and follow with
desirable features. Much of what I describe below is generic and applies to
test suites written in any programming language, despite many examples being
written in Perl. As I get further along, I'll start discussing
[Test::Class::Moose](https://github.com/Ovid/test-class-moose), a testing
framework written in Perl. While the concepts will still generally be generic,
the implementation examples will be in Perl.

I often speak with developers who take a new job and they describe a Web site
built out of a bunch of `.pl` scripts scattered randomly through directories,
lots of duplicated code, poor use of modules, with embedded SQL and printing
HTTP headers and HTML directly. The developers shake their head in despair,
but grudgingly admit an upside: job security. New features are time-consuming
to add, changes are difficult to implement and may have wide-ranging
side-effects, and reorganizing the code base to have a proper separation of
concerns, to make it cheaper and safer to hack on, will take lots and lots of
time.

A bunch of randomly scattered scripts, no separation of concerns, lots of
duplicated code, poor use of modules, SQL embedded directly in them? Does this
sound familiar? It's your standard Perl test suite. We're horrified by this in
the code, but don't bat an eyelash at the test suite.

Part of this is because much, if not most, of the Perl testing culture focuses
on testing distributions, not applications. If you were to look at the tests
for my module
[DBIx::Class::EasyFixture](https://github.com/Ovid/dbix-class-easyfixture),
you'd see the following tests:

    00-load.t
    basic.t
    definitions.t
    groups.t
    load_multiple_fixtures.t
    many_to_many.t
    no_transactions.t
    one_to_many.t
    one_to_one.t

These tests were added one by one, as I added new features to
`DBIx::Class::EasyFixture` and each `*.t` file represents (more or less) a
different feature.

For a small distribution, this isn't too bad because it's very easy to keep it
all in your head. With only 9 files, it's trivial to glance at them, or grep
them, to figure out where the relevant tests are. Applications, however, are a
different story. This is from one of my customer's test suites:

    $ find t -type f | wc -l
    288

That's actually fairly small. One code base I worked on had close to a million
lines of code with thousands of test scripts. You couldn't hold the code base
in your head, you're couldn't *glance* at the the tests to figure out what
went where, nor was grepping necessarily going to tell you as tests for
particular sections of code were often scattered around multiple test scripts.
And, of course, I regularly heard the lament that I've heard at many shops
with larger code bases: where are the tests for feature *X*? Instead of just
sitting down and writing code, the developers are hunting for the tests,
wondering if there are any, and, if not, trying to figure out where to put
their new tests.

Unfortunately, this disorganzation is only the start of the problem.

## Large-scale test suites

I've worked with many companies with large test suites and they tend to
share some common problems. I list them in below in roughly the order I
try to address these problems (in other words, from easiest to hardest).

* Tests often emit warnings
* Tests often fail ("oh, that sometimes fails. Ignore it.")
* There is little evidence of organization
* Much of the testing code is duplicated
* Testing fixtures are frequently not used (or poorly used)
* They take far too long to run
* Code coverage is spotty

Problems are one thing, but what features do we want to see in large-scale
test suites?

* Tests should be very easy to write and run
* They should run relatively quickly
* The order in which tests run should not matter
* Test output should be clean
* It should be obvious where to find tests for a particular piece of code
* Testing code should not be duplicated
* Code coverage should be able to analyze different aspects of the system

But first, let's take a look at some of the problems and try to understand
their impacts. While it's good to push a test suite into a desirable state,
often this is risky if the underlying problems are ignored. I will offer
recommendations for resolving each problem, but it's important to understand
that these are *recommendations*. They may not apply to your situation.

### Tests often emit warnings

This seems rather innocuous. Sure, code emits warnings and we're used to that.
Unfortunately, we sometimes forget that warnings are *warnings*: there might
very well be something wrong. In my time at the BBC, one of the first things I
did was try to clean up all of the warnings. One was a normal warning about
use of an undefined variable, but it was unclear to me from the code if this
should be an acceptable condition. Another developer looked at it with me and
realized that the variable should never be undefined: this warning was masking
a very serious bug in the code, but the particular condition was not
explicitly tested. By rigorously eliminating all warnings, we found it easier
to make our code more correct, and in those places where things were dodgy,
comments were inserted into the code to explain why warnings were suppressed.
In short: the code became easier to maintain.

Another issue with warnings in the test suite is that they condition
developers to ignore warnings. We get so used to them that we stop reading
them, even if something serious is going on (on a related note, I often listed
to developers complain about stack traces, but a careful reading of a stack
trace will often reveal the exact cause of the exception). New warnings crop
up, warnings change, but developers conditioned to ignore them often overlook
serious issues with their code.

**Recommendation**: Eliminate all warnings from your test suite, but
investigate each one to understand if it reflects a serious issue. Also, some
tests will capture STDERR, effectively hiding warnings. Making warnings fatal
while running tests can help to overcome this problem.

### Tests often fail ("oh, that sometimes fails. Ignore it.")

For one client, their hour-long test suite had many failing tests. When I
first started working on it, I had a developer walk me through all of the
failure and explain why they failed and why they were hard to fix. Obviously
this is a far more serious problem than warnings, but in the minds of the
developers, they were under constant deadline pressures and as far as
management was concerned, the test suite was practically a luxury, not
"serious code." As a result, developers learned to recognize these failures
and consoled themselves with the thought that they understood the underlying
issues.

Of course, that's not really how it works. The developer explaining the test
failures admitted that he didn't understand some of them and with longer test
suites that routinely fail, more failures tend to crop up, but developers
conditioned to accept failures tend not to notice them. They kick off the test
suite, run and grab some coffee and later glance over results to see if they
look reasonable (that's assuming they run all of the tests, something which
often stops happening at this point). What's worse, continuous integration
tools are often built to accomodate this. From the Jenkin's [xUnit Plugin
page](https://wiki.jenkins-ci.org/display/JENKINS/xUnit+Plugin):

> # Features
>  * Records xUnit tests
>  * Mark the build unstable or fail according to threshold values

In other words, there's an "acceptable" level of failure. What's the
acceptable level of failure when you debit someone's credit card, or you're
sending their medical records to someone, or you're writing embedded software
that can't be easily updated?

Dogmatism aside, you can make a case for acceptable levels of test failure,
but you need to understand the risks and be prepared to accept them. However,
for the purposes of this document, we'll assume that the acceptable level of
failure is zero.

If you absolutely cannot fix a particular failure, you should at least mark
the test as `TODO` so that the test suite can pass. Not only does this help to
guide you to a clean test suite, the `TODO` reason is generally embedded in
the test, giving the next developer a clue what's going on.

**Recommendation**: Do not allow any failing tests. If tests fail which do not
impact the correctness of the appliaction (such as documentation or "coding style"
tests), they should be separated from your regular tests in some manner and
your systems should recognize that it's OK for them to fail.

### There is little evidence of organization

As mentioned previously, a common lament amongst developers is the difficulty
of finding tests for the code they're working on. Consider the case of
[HTML::TokeParser::Simple](http://search.cpan.org/dist/HTML-TokeParser-Simple/).
The library is organized like this:

    lib/
    └── HTML
        └── TokeParser
            ├── Simple
            │   ├── Token
            │   │   ├── Comment.pm
            │   │   ├── Declaration.pm
            │   │   ├── ProcessInstruction.pm
            │   │   ├── Tag
            │   │   │   ├── End.pm
            │   │   │   └── Start.pm
            │   │   ├── Tag.pm
            │   │   └── Text.pm
            │   └── Token.pm
            └── Simple.pm

There's a class in their named
`HTML::TokeParser::Simple::Token::ProcessInstruction`. Where, in the following
tests, would you find the tests for process instructions?

    t
    ├── constructor.t
    ├── get_tag.t
    ├── get_token.t
    ├── internals.t
    └── munge_html.t

You might think it's in the `get_token.t` test, but are you sure? And
what's that strange `munge_html.t` test? Or the `internals.t` test? As
mentioned, for a small library, this really isn't too bad. However, what if we
reorganized our tests to reflect our library hierarchy?

    t/
    └── tests/
        └── html/
            └── tokeparser/
                ├── simple/
                │   ├── token/
                │   │   ├── comment.t
                │   │   ├── declaration.t
                │   │   ├── tag/
                │   │   │   ├── end.t
                │   │   │   └── start.t
                │   │   ├── tag.t
                │   │   └── text.t
                │   └── token.t
                └── simple.t 

It's clear that the tests for `HTML::TokeParser::Simple::Token::Tag::Start`
that the tests are in `t/tests/html/tokeparser/simple/token/tag/start.t`. And
you can see easily that there is no file for `processinstruction.t`. This test
organization not only makes it easy to find where your tests are, it's also
easy to program your editor to automatically switch between the code and the
tests for the code. For large test suites, this saves a huge amount of time.
When I reorganized the test suite of the BBC's central metadata repository,
[PIPs](http://www.bbc.co.uk/blogs/bbcinternet/2009/02/what_is_pips.html), I
followed a similar pattern and it made our life much easier.

Of course, your test suite could easily be more complicated and your top-level
directories inside of your test directory may be structured differently:

    t
    ├── unit/
    ├── integration/
    ├── api/
    └── web/

**Recommendation**: Organize your test files to have a predictable,
discoverable structure. The test suite should be much easier to work with.

### Much of the testing code is duplicated

We're aghast that that people routinely cut-n-paste their application code,
but we don't even notice when people do this in their test code. More than
once I've worked on a test suite with a significant logic change and I've had
to find this duplicated code and either change it many places or try to
refactor it so that it's in a single place and then change it. We already know
why duplicated code is bad, I'm unsure why we tolerate this in test suites.

Much of my work in tests has been to reduce this duplication. For example,
many test scripts lists the same set of modules at the top. I did a heuristic
analysis of tests on the CPAN and chose the most popular testing modules and
that allowed me to change this:

    use strict;
    use warnings;
    use Test::Exception;
    use Test::Differences;
    use Test::Deep;
    use Test::Warn;
    use Test::More tests => 42;

To this:

    use Test::Most tests => 42;

You can easily use similar strategies to bundle up common testing modules
into a common testing module that all of your tests use. Less boilerplate and
you can easily dive into testing.

Or as a more egregious example, I often see something like this:

    set_up_some_data($id);
    my $object = Object->new($id);

    is $object->attr1, $expected1, 'attr1 works';
    is $object->attr2, $expected2, 'attr2 works';
    is $object->attr3, $expected3, 'attr3 works';
    is $object->attr4, $expected4, 'attr4 works';
    is $object->attr5, $expected5, 'attr5 works';

And then a few lines later:

    set_up_some_data($new_id);
    $object = Object->new($new_id);

    is $object->attr1, $new_expected1, 'attr1 works';
    is $object->attr2, $new_expected2, 'attr2 works';
    is $object->attr3, $new_expected3, 'attr3 works';
    is $object->attr4, $new_expected4, 'attr4 works';
    is $object->attr5, $new_expected5, 'attr5 works';

And then a few lines later, the same thing ...

And in another test file, the same thing ...

Put that in its own test function and wrap those attribute tests in a loop. If
this pattern is repeated in different test files, put it in a custom test
library:

    sub test_fetching_by_id {
        my ( $class, $id, $tests ) = @_;
        my $object = $class->new($id);

        # this causes diagnostics to display the file and line number of the
        # caller on failure, rather than reporting *this* file and line number
        local $Test::Builder::Level = $Test::Builder::Level + 1;

        foreach my $test (@$tests) {
            my ( $attribute, $expected ) = @$test;
            is $object->$attribute, $expected, "$attribute works";
        }
    }

And then you call it like this:

    my %id_tests = (
        $id => [
            [ attr1 => $expected1 ],
            [ attr2 => $expected2 ],
            [ attr3 => $expected3 ],
            [ attr4 => $expected4 ],
            [ attr5 => $expected5 ],
        ],
        $new_id => [
            [ attr1 => $new_expected1 ],
            [ attr2 => $new_expected2 ],
            [ attr3 => $new_expected3 ],
            [ attr4 => $new_expected4 ],
            [ attr5 => $new_expected5 ],
        ],
    );

    while ( my ( $id, $tests ) = each %id_tests ) {
        test_fetching_by_id( 'Object', $id, $tests );
    }

This is a cleanly refactored data-driven approach. By not repeating yourself,
if you need to test new attributes, you can just add an extra line to the data
structures and the code remains the same. Or, if you need to change the logic,
you know you only have one spot in your code where this is done. Once a
developer understands the `test_fetching_by_id()`, then can reuse this
understanding in multiple places. Further, it makes it easier to find patterns
in your code and any competent programmer is always on the lookout for
patterns because those are signposts leading to cleaner designs.

**Recommendation**: Keep your test code as clean as your application code.

### Testing fixtures are frequently not used (or poorly used)

One difference between your application code and the test suite is in an
application, we often have no idea what the data will be and we try to have a
clean separation of data and code.

In your test suite, we also want a clean separation of data and code (in my
experience, this is very hit-or-miss), but we often *need* to know the data we
have. We set up data to run tests against to ensure that we can test various
conditions. Can we give a customer a birthday discount if they
were born February 29th? Can a customer with an overdue library book check out
another? If our employee number is no longer in the database, is our code
properly deleted, along with the backups? (kidding!)

When we set up the data for these known conditions under which to test, we
call the data a [test fixture](http://en.wikipedia.org/wiki/Test_fixture).
Test fixtures, when properly designed, allow us generate clean,
understandable tests and make it easy to write tests for unusual conditions
that may otherwise be hard to analyze.

There are several common anti-patterns I see in fixtures.

* Hard to set up and use
* Adding them to the database and not rolling them back
* Loading all your test data at once with no granularity

In reviewing various fixture modules on the CPAN and for clients I have worked
with, much of the above routinely holds true. On top of that, documentation is
often rather sparse or non-existent. Here's a (pseudo-code) example of an
almost undocumented fixture system for one client I worked with and it
exemplified common issues in this area.

    load_fixture(
        database => 'sales',
        client   => $client_id,
        datasets => [qw/ customers orders items order_items referrals /],
    );

This had several problems, all of which could be easily corrected *as code*,
but they built a test suite around these problems and had backed themselves
into a corner, making their test suite dependent on bad behavior.

The business case is that my client had a product serving multiple customers
and each customer would have multiple separate databases. In the above,
client *$client_id* connects to their sales database and we load several test
datasets and run tests against them. However, loading of data was not done in
a transaction, meaning that there was no isolation between different test
cases in the same process. More than once I caught issues where running an
individual test case would often fail because it depended on data loaded by a
different test case, but it wasn't always clear which test cases were coupled
with which.

Another issue is that fixtures were not fine-tuned to address particular test
cases. Instead, if you loaded "customers" or "referrals", you got *all* of
them in the database. Do you need a database with a single customer with a
single order and only one order item on it to test that obscure bug that
occurs when a client first uses your software? There really wasn't any clean
way of doing that; data was loaded in an "all or nothing" context. Even if you
violated the paradigm and tried to create fine-tuned fixtures, it was very,
very hard to write them due to the obscure, undocumented format needed to
craft the data files for them.

Because transactions were not used and changes could not be rolled back, each
`*.t` file would rebuild its own test database, a very slow process. Further,
due to lack of documentation about the fixtures, it was often difficult to
figure out which combination of fixtures to load to test a given fixture. Part
of this is simply due to the complex nature of the business rules, but the
core issues stemmed from a poor understanding of fixtures. This client now has
multiple large, slow test suites, spread across multiple repositories, all of
which constantly tear down and set up databases and load large amounts of
data. The test suites are both slow and fragile The time and expense to fix
this problem is considerable due to how long they've pushed forward with this
substandard setup.

What you generally want is the ability to easily create understandable
fixtures which are loaded in a transaction, tests are run, and then changes
are rolled back.  The fixtures need to be fine-grained so you can tune them
for a particular test case.

One attempt I've made to fix this situation is releasing
[DBIx::Class::EasyFixture](http://search.cpan.org/dist/DBIx-Class-EasyFixture/lib/DBIx/Class/EasyFixture.pm),
along with [a tutorial](http://search.cpan.org/dist/DBIx-Class-EasyFixture/lib/DBIx/Class/EasyFixture/Tutorial.pm).
It does rely on `DBIx::Class`, the most popular ORM for Perl. This will likely
make it unsuitable for some uses cases.

Using them is very simple:

    my $fixtures = DBIx::Class::EasyFixture->new(schema => $schema);
    $fixtures->load('customer_with_order_without_items');

    # run your tests

    $fixtures->unload; # also unloads when out of scope

For the customer's code, we could satisfy the different database requirements
by passing in different schemas. Other (well-documented) solutions,
particularly those which are pure `DBI` based are welcome in this area.

**Recommendation**: Fine-grained, well-documented fixtures which are easy to
create and easy to clean up.

### They take far too long to run

The problem with long-running test suites is well known, but it's worth
covering this again here. These are problems that others have discussed and
that I have also personally experienced many times.

![Perl's version of waiting for a compile](http://i.imgur.com/JNfyxoo.png)

*With apologies to [XKCD](http://xkcd.com/303/)*

In the best case scenario for a long-running test suite, expensive developer
time is wasted while the test suite is running. When they launch that
hour-long (or more) test suite, they frequently take a break, talk to (read:
interrupt) other developers, check their Facebook, or do any number of things
which equate to "not writing software." Yes, some of those things involve
meetings or research, but meetings don't conveniently schedule themselves when
we run tests and for mature products (those which are more likely to have
long-running test suites), there's often not that much research we really need
to do.

Here are some of the issues with long-running test suites:

* Expensive developer time is wasted while the test suite runs
* Developers often don't run the entire test suite
* Code is fragile as a result

What I find particularly curious is that we accept this state of affairs. Even
a back-of-the-envelope calculation can quickly show significant productivity
benefits that will pay off in the long run by taking care of our test suite.
[I once reduced a test suite's run time from one hour and twenty minutes down
to twelve
minutes](http://www.slideshare.net/Ovid/turbo-charged-test-suites-presentation)
(*Note: today I use a saner approach that results in similar or greater
performance benefits*).  We had six developers on that team. When the test
suite took over an hour to run, they often didn't run the test suite. They
would run tests on their section of code and push their code when they were
comfortable with it. This led to other developers finding buggy code and
wasting time trying to figure out how they broken it when, in fact, someone
else broke the code.

But let's assume each developer was running the test suite at least once a
day. By cutting test suite run time by over an hour, we reclaimed a *full day*
of developer productivity every day! Even if it takes a developer a month to
increase perfomance by that amount it pays for itself many times over very
quickly. Why would you not do this?

There are several reasons why this is difficult. Tasking a developer with a
block of time to speed up a test suite means the developer is not creating
user-visible features during that time. For larger test suites, it's often
impossible to know in advance just how much time you can save or how long it
will take you to reach your goal. In most companies I've worked with, the
people who can make the decision to speed up the test suite are often not the
people feeling the pain. Productivity and quality decrease slowly over time,
leading to the [boiling frog
problem](http://en.wikipedia.org/wiki/Boiling_frog).

What's worse: in order to speed up your test suite without affecting behavior,
the test suite often has to be "fixed" (eliminating warnings, failures, and
reducing duplication) to ensure that no behavior has been changed during the
refactor.

Finally, some developers simply don't have the background necessary to
implement performance optimizations. While performance profiles such as Perl's
[Devel::NYTProf](http://search.cpan.org/dist/Devel-NYTProf/lib/Devel/NYTProf.pm)
can easily point out problem areas in the code, it's not always clear how to
overcome the discovered limitations.

The single biggest factor in poor test suite performance for applications is
frequently I/O. In particular, working with the database tends to be a
bottleneck and there's only so much database tuning that can be done. After
you've profiles your SQL and optimized it, several database-related
optimizations which can be considered are:

1. Using transactions to clean up your database rather than rebuilding the
   database
2. Only connect to the database once per test suite (hard when you're using
   separate processes
3. If you must rebuild the database, maintain a pool of test databases and
   assign them as needed, rebuilding used ones in the background
4. Use smaller database fixtures instead of loading everything at once

After you've done all you can to improve your database access, you may find
that your test suite is "fast enough", but if you wish to go further, there
are several steps you can take.

#### Use [Test::Aggregate](http://search.cpan.org/dist/Test-Aggregate/)

`Test::Aggregate` can often double the speed of your test suite (I've had it
speed up test suites by around 65%). It does this by taking your separate
`*.t` files and runs them in a single process. Not all tests can be run this
way (tests that munge global state without cleaning up are prime examples),
but it's the easiest way to get a quick boost to test suite performance.

#### Aggressively search for and remove duplicated tests.

For poorly organized test suites, developers sometimes make the mistake of
putting tests for something in a new `*.t` file or add them to a different
`*.t` file, even if related tests already exist. This strategy can be
time-consuming and often does not result in quick wins.

#### Use `Devel::NYTProf` aggressively.

For one test suite, I found that we were using a pure Perl implementation of
JSON. As the test suite used JSON extensively, switching to
[JSON::XS](http://search.cpan.org/dist/JSON-XS/XS.pm) gave us a nice
performance boost. We may not have noticed that if we hadn't been profiling
our code.

#### Inline "hot" functions.

Consider the following code which runs in about 3.2 seconds on my computer:

    #!/usr/bin/env perl

    use strict;
    use warnings;
    no warnings 'recursion';

    for my $i ( 1 .. 40 ) {
        for my $j ( 1 .. $i**2 ) {
            my $y = factorial($j);
        }
    }

    sub factorial {
        my $num = shift;
        return 1 if $num <= 1;
        return $num * factorial($num - 1);
    }

By rewriting the recursive function as a loop, the code takes about .87
seconds:

    sub factorial {
        my $num = shift;
        return 1 if $num <= 1;
        $num *= $_ for 2 .. $num - 1;
        return $num;
    }

By inlining the calculation, the code completes in .69 seconds:

    for my $i ( 1 .. 40 ) {
        for my $j ( 1 .. $i**2 ) {
            my $y = $j;
            if ( $y > 1 ) {
                $y *= $_ for 2 .. $y - 1;
            }
        }
    }

In other words, in our trivial example, the inlined behavior is roughly 20%
faster than the iterative function and 80% faster than the recursive function.

#### Recompile your Perl

You may wish to recompile your Perl to gain a performance improvement. Many
Linux distributions ship with a threaded Perl by default. Depending on the
version of Perl you ship with, you can gain performance improvements of up to
30% by recompiling without threads. Of course, if you use threads, you'll feel
very stupid for doing this. However, if you don't make heavy use of threads,
switching for a forking model for the threaded code may make the recompile
worth it. Naturally, you'll need to heavily benchmark your code (preferably
under production-like loads) to understand the trade-offs here.

#### Preload modules

#### Parallel tests

#### Devel::CoverX::Covered

### Code coverage is spotty
