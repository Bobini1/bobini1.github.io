---
layout: post
title:  "Parsing a tag-based file format with nested blocks (with PEGTL)"
date:   2022-08-06 13:21:00 +0200
categories: cpp
---

## Background

A big and important challenge for any community-driven rhythm game is parsing *charts* (AKA files defining what the player is going to play).
Over the years, many formats emerged: BMS, PMS, DWI, SM, SSC, KSH, and so on, and so on. Most free rhythm games are based on commercial titles: Dance With Intensity and StepMania are based on Dance Dance Revolution, K-Shoot Mania originates from Sound Voltex, BM98 was inspired by Beatmania.

The chart formats of the original games were not public, obviously, and even if they were, I'm not certain whether they would be fitting for user creations. And so, each community game tried to define what is necessary to enable their players to create content.

## The first challenges

One of the first such attempts was made by Urao Yane, a Japanese developer who wanted to help others learn how to play instruments by giving them a game similar to Beatmania.
In 1998, together with NBK, he released BM98 to the world.

I decided that, among others, I wanted to support his *Be-Music Script* (.bms) file format in my own rhythm game.

The format is based on two basic types of tags:

# One-line single-value tags

    #TITLE A Beautiful Sky
    #Artist recognize-m
    #BPM 137

Those are the simplest to parse, every tag is in its own line and strings with spaces don't need to be quoted.
Even the simplest parser can go through them line-by-line.

# Channel-based tags

    #00112:00220000
    #00113:00003300
    #00114:00000044

Channel-based tags represent events that have particular timings.

    #00112:00220000
     ^^^

The first three digits are the measure number. With 4/4 meter (default), every measure contains four beats. With `#BPM 137`, there would be around 34 measures per minute.
The measure number specifies which time segment is defined by this particular line.

    #00112:00220000
        ^^

The next two digits specify the *channel* - that's the actual type of the command.
BMS is a game where rectangles fall in 6 lanes (or 12 when playing double mode) - channel `12` means *column 2 of player 1 side*.
There are many other channels: `01` is used for *background notes*, `03` is for BPM changes, `04` is for background animations, etc., etc.

    #00112:00220000
           ^^^^^^^^

Finally, last is the measure definition. It lists all events that happen in the particular measure and channel.
The definition is divided into pairs, in this case `00`, `22`, `00` and `00`.
`00` is simply a gap where nothing happens, while `22` is a reference to a definition, e.g. `#WAV22 cymbal.wav`.
When the player successfully hits the rectangle on lane two at the second quarter of the first measure, the cymbal sound is supposed to play.

Overall, the basic ideas are not too complicated. The format is pretty much human-readable and doesn't seem to have too many gotchas.

## The gotcha

Life isn't so easy, sadly.

In the original format, there were exactly also exactly two **block tags**, always used together.

    #RANDOM 2
    #IF 1
    #00112:00220000
    #ENDIF
    #IF 2
    #00112:00220000
    #ENDIF
    #ENDRANDOM

The `#RANDOM` tag can specify a portion of the file that can change every time the song is played.
A number is rolled and an `#IF` tag is selected.

And it can contain *anything*.

This is a big problem. Notes can be random, but sound file definitions can be random too. The title can be random, BPM changes can be added randomly.

And, worst of all, *an `#IF` can contain* ***even more*** *random blocks*.

So now we end up with an indeterministic format that pretty much blocks any pre-processing that we may want to do. Offsets can't be pre-calculated, files can't be cached easily.
Anything with `#RANDOM` automatically requires extra handling.

Pain.

## Parsing that mess

I really, really wanted to avoid writing a parser manually. They don't scale at all and are impossible to understand from the outside because of all the implementation details leaking to the grammar definitions.
I wanted to avoid creating the parser at runtime as well, those can get heavy and it's better to avoid long loading times when you can.

Therefore, I went on a journey to find something fitting my needs.

# Previous experiences with parsing

I've used the [nom](https://github.com/Geal/nom) library while learning Rust some time ago. It was a pleasure and I didn't expect that it would be a challenge to find something like that in C++, which is the more popular language, after all.
Well, I was wrong. Too bad.

# Flex/Bison

Bison was instantly out of the question because I wanted to keep my project MIT-licensed.

Flex is extremely old, requires a separate configuration step, and requires learning an entire new syntax to use (it has its own file format for grammars).

I decided to consider it only if everything else fails.

# lexy

Lexy had chosen a different path. Instead of separate grammar files that would be used for code generation, its parsers were defined directly in source code and generated at compile time. Clean and good.

On paper, lexy seemed extremely nice. It was a bit convoluted but I guess you can't really avoid that for such a complicated task.

However, it turned out to be a very, very fresh library (the first official release was only a month before). It was obvious that I wouldn't be able to find too much help if I ran into problems with it. It was pretty much the opposite extreme to Flex, modern but untested in practice.

I was reluctant to adopt it.

# PEGTL

Lexy listed PEGTL as one of its inspirations. Curious, I checked it out.

I wouldn't say that I fell in love with it but I was intrigued. It was very unusual but at the same time not too complicated in its syntax.
In essence, it uses inheritance from variadic template classes for composing parsers. The documentation was a bit cryptic but there was a ton of examples.

This is what I went for in the end. It was both battle-tested (created ~10 years ago) and modern.

## Implementation

It turned out that [PEG parsers](https://en.wikipedia.org/wiki/Parsing_expression_grammar) aren't too good at parsing trees. If it wasn't for `#RANDOM`, the implementation would be almost trivial but there was nothing I could do, my parser couldn't be incomplete. So I just got to work.

{% highlight C++ %}

template<typename AllowedValue, typename TagName>
struct MetaTag
  : pegtl::seq<pegtl::star<pegtl::space>,
               pegtl::one<'#'>,
               TagName,
               pegtl::star<pegtl::space>,
               AllowedValue,
               pegtl::star<pegtl::minus<pegtl::space, pegtl::eolf>>>
{
};

{% endhighlight %}

Since there were many types of tags but all had a similar form, I designed a template to do some of the work for me.
`AllowedValue` and `TagName` were also parsers, of course.

{% highlight C++ %}

struct SpacesUntilEndOfLine
  : pegtl::seq<pegtl::star<pegtl::minus<pegtl::space, pegtl::eolf>>,
               pegtl::at<pegtl::eolf>>
{
};

struct MetaString : pegtl::until<SpacesUntilEndOfLine>
{
};

{% endhighlight %}

Most single-value tags were text-based so a simple parser matching any string would be enough for an `AllowedValue`, in most cases.

# Assigning tokens (the first "fun" part)

In order to provide a "callback" action class to pegtl, which would assign matched data to some data structure, it needs to be bound to a parser class that is used in the final composition of parsers.

If I bound an action to the `Title` parser, I would get the title... But also the `#TITLE` text and any extra spaces around it! Alternatively, I could bind an action to just the `MetaString` that `Title` uses but then I wouldn't be able to know which actual tag it came from, since they all use `MetaString`'s - I wouldn't be able to assign the matched string to anything specific!

The outlook was dire.

However, it was then that I realized that my problems would go away if I directly inherited `MetaString` to something like `TitleMetaString` directly and used that in my `Title` parser. That solution was pretty much perfect, if not for one small issue.

That would be *so much typing*.

Imagine it, I would need to make a `MetaString` duplicate, then create an `Action` specialization for that duplicate and then define a struct for the entire tag to use it in pretty much the same way every time. Just for your information, BMS has approximately 70 different tags. I wouldn't be able to stand that.

At that point, I got a crazy idea.

**Macros!**

They are so hated around but in this particular case, they were a godsend. It took me some time but I finally came up with this beauty:

{% highlight C++ %}
#define RHYTHMGAME_TAG_PARSER(                                                 \
  tag, tagstrlen, allowedValue, memberFnPointer, parser)                       \
    struct tag##_allowedValue : allowedValue                                   \
    {                                                                          \
    };                                                                         \
                                                                               \
    struct tag                                                                 \
      : MetaTag<tag##_allowedValue,                                            \
                pegtl::istring<RHYTHMGAME_TO_CHARS(tagstrlen, #tag)>>          \
    {                                                                          \
    };                                                                         \
                                                                               \
    template<>                                                                 \
    struct Action<tag##_allowedValue>                                          \
    {                                                                          \
                                                                               \
        template<typename ActionInput>                                         \
        static auto apply(const ActionInput& input, TagsWriter& chart) -> void \
        {                                                                      \
            CALL_MEMBER_FN(chart, memberFnPointer)((parser)(input.string()));  \
        }                                                                      \
    };
{% endhighlight %}

Used like this:

{% highlight C++ %}
RHYTHMGAME_TAG_PARSER(SubTitle, 8, MetaString, &TagsWriter::setSubTitle, trimR);
RHYTHMGAME_TAG_PARSER(Bpm, 3, Floating, &TagsWriter::setBpm, std::stod);
{% endhighlight %}

`RHYTHM_GAME_TO_CHARS` is a small helper macro I got from Stack Overflow which splits a string into chars that can be used as a variadic template argument.
(The integer argument is something I couldn't avoid, sadly, but I still like it more than `pegtl::istring<'S', 'U', 'B', 'T', 'I', 'T', 'L', 'E'>`).
You can check out the source [here](https://github.com/Bobini1/RhythmGame/blob/493b1934e529bc8f16544c43b8dd66e79df65c43/src/charts/chart_readers/ToChars.h).

The entire macro has saved me from writing hundreds of lines of code so I consider it a success.

# Tree (the second "fun" part)

After talking to some helpful people at the [C++ Slack server](https://join.slack.com/t/cpplang/shared_invite/zt-15acmok1h-ZxmQgn06jCcw8ajanVa4Rg),
I finally got an idea how to create the `#RANDOM`-`#IF` tree. (The PEGTL Discord server is dead, don't even bother looking for help there).

It was basically another manual job which I couldn't even automate too this time. I just had to make a stack and manage popping and pushing when entering and leaving subsequent blocks.

{% highlight C++ %}

struct Tags
{
    std::optional<std::string> title;
    std::optional<std::string> artist;
    std::optional<double> bpm;
    std::optional<std::string> subTitle;
    std::optional<std::string> subArtist;
    std::optional<std::string> genre;

    // we have to use std::unique_ptr<std::multimap> because otherwise this
    // doesn't compile on MSVC. :)
    std::vector<
        std::pair<RandomRange, std::unique_ptr<std::multimap<IfTag, Tags>>>>
        randomBlocks; /*< Random blocks can hold any tags, including ones that
                        were already defined. */
};
{% endhighlight %}

I wrote a struct for holding stuff (yes, the data structure for random blocks just has to look like that).

{% highlight C++ %}

class TagsWriter
{
    std::stack<std::pair<IfTag, Tags>> ifStack;
    std::stack<std::pair<RandomRange, std::vector<std::pair<IfTag, Tags>>>>
      randomStack;

  public:
    TagsWriter() { ifStack.push(std::make_pair(0, Tags{})); }
    void setTitle(std::string title)
    {
        ifStack.top().second.title = std::move(title);
    }

    ...

    void setBpm(double bpm) { ifStack.top().second.bpm = bpm; }

    void enterIf(IfTag tag) { ifStack.push(std::make_pair(tag, Tags{})); }

    void leaveIf()
    {
        randomStack.top().second.emplace_back(ifStack.top().first,
                                              std::move(ifStack.top().second));
        ifStack.pop();
    }

    void enterRandom(RandomRange range)
    {
        randomStack.push(
          std::make_pair(range, std::vector<std::pair<IfTag, Tags>>{}));
    }

    void leaveRandom()
    {
        auto ifs = std::move(randomStack.top().second);
        auto randomDistribution = randomStack.top().first;
        randomStack.pop();
        auto ifsMap = std::multimap<IfTag, Tags>{};
        for (auto& ifTag : ifs) {
            ifsMap.emplace(ifTag.first, std::move(ifTag.second));
        }
        ifStack.top().second.randomBlocks.emplace_back(std::make_pair(
          randomDistribution,
          std::make_unique<std::multimap<IfTag, Tags>>(std::move(ifsMap))));
    }

    auto getTags() -> Tags& { return ifStack.top().second; }
};

{% endhighlight %}

The `TagsWriter` is a wrapper around `Tags` which takes care of the two stacks.

As you can see in the code, the methods of `TagsWriter` are a bit complex but in reality, the principle is simple.
When we enter a `#RANDOM` or `#IF`, we push it to the appropriate stack, initially empty.
We then continue to write to the current top `#IF`. Once we leave it, we pop it from the `ifStack` and add it to the current top `#RANDOM` block.
And once we leave that, we pop it and add it to the `#IF` it was inside of.

For convenience, the outer block of the tree (the "root") is treated as an `#IF` block (as you can see in the constructor and, consequently, `getTags()`).

## Afterword and future challenges to tackle

I must say that solving these problems was very challenging but also quite rewarding.
It's a decent base that I can use for further development, despite being a bit rough around the edges still.
Undeniably, the parser is crazy fast and I should be able to extend it without any issues. PEGTL seems to have been a good choice.

The problem of calculating time offsets remains - it may be difficult with all the degrees of freedom
(the BPM can change anytime, the meter can change almost any time, the time subdivisions inside measures can be anything).

All in all though, I think the most difficult task in regard to parsing BMS has been accomplished.

Thanks for reading! This was my first post here. : )
