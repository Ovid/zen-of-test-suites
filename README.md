# The Zen of Test Suites

Serious testing for serious software

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
often this is risky if the underlying problems are ignored.

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

It's clear that the tests for `HTML::TokeParser::Simple::Token::Tag::Start` that
the tests are in `t/tests/html/tokeparser/simple/token/tag/start.t`. And you can
see easily that there is not file for `processinstruction.t`. This test
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

So long as the test directories in those top-level directories follow a
predictable, discoverable structure, the test suite should be much easier to
work with.

### Much of the testing code is duplicated



### Testing fixtures are frequently not used (or poorly used)

### They take far too long to run


### Code coverage is spotty


