+++
date = '2025-09-12T8:00:00+08:00'
draft = false
title = 'English for Programmers - 02'
categories = ['Blog']
tags = ['English']
+++

# Unit 2. Code Review & Testing

this unit will cover:

- Differentiate between various **testing strategies**
- Write **professional guidelines**
- Sound more natural and smooth when **asking questions**
- Use colloquial language in speaking to **give and accept feedback**

## vocabulary - None Pharses 词汇: 名词短语

Testing 测试是软件开发中的重要短语, 用于检查软件是否达到特定的标准和用户需求.

使用这些名词语句来证明英语名词的专业性:

| None                    | Desciption                                                  | Example                                                         |
| :---------------------- | :---------------------------------------------------------- | :-------------------------------------------------------------- |
| time box                | an allocated period of time for completing a task           | - allcate a 2-hour time box for regression testing              |
| stress test             | a method to assess a system's performance under heavy loads | - simulate 1000 users accessing the login page at the same time |
| sanity check 健全性测试 | a quick check to verify that something is as expected       | - are the units of the output value correct?                    |
| ad hoc test 临时测试    | a test performed without predefined test cases or plans     | - input unexpected characters into a search bar                 |
| edge case               | a problem that only happens in extreme situations           | - upload an empty, 0-byte file                                  |

一些名词短语也可以当作动词使用:

- Can you _sanity check_ my email before I send it? I want to make sure there aren't any errors.
- We have a lot to do today. Let's _time box_ this meeting so we stay on sechedule.

> BE CAREFUL: sanity check vs ad hoc test  
> _sanity check_: similar to performing a review  
> _ad hoc test_: exploring issues using the tester's knowledge

- Yesterday, while doing **ad hoc test**, I came across some unexpected behaviour when I was randomly interacting with the system.
- Hey, I made a few changes to the codebase. Can you do a quick **sanity check** before I start testing?

## Parallel Structure 平行结构

Parallel structure: Using the same grammatical structure for two or more clauses in a sentence.

Example:  
"In our coding guidelines, we emphasise **writing** clear comments, **to follow** naming conventions, and **maintain** consistent indentation."

注意到上面的动词了吗? (mixed verb forms)  
**writing** = gerund form 动名词形式
**to follow** = infinitive form 不定式形式
**maintain** = base form

为了达到平行结构, 将动词修改为相同形式. 由于上面例子由 'emphasise' 开始, 将其改成 **-ing** 形式:  
In our coding guidelines, we emphasise **writing** clear comments, **following** naming conventions, and **maintaining** consistent indentation.

平行结构的好处:

- more professional
- more effective
- easier to read and follow

You can apply this technique when writing:

- documentation
- code comments
- best practices guidelines

## Connected Speech 连贯的语言

When we talk in everyday conversations, our words shouln't stand alone.

Instead, some sound, words and phrases are merged together in what's called connected speech.
It's a natural _rhythm_ and _flow_ taht make conversations sound more smooth.

Let's take a closer look at three different techniques that your can use while asking questions:

- 同化 Assimilation  
  将两个音素融合形成新音素.  
  如"could you"中/d/与/j/融合发成/dʒ/, 形成"coujoo"的流畅发音. /d/ + /y/ = /dʒ/

- 省略 Reduction  
  缩短或省略特定音素.  
  如"who is"中/oʊ/和/ɪ/压缩为长元音/uː/, 读作"hooz"

- 连读 Linking  
  将前词尾音与后词首音无缝连接，无停顿.  
  如"how about"中/w/与/ə/自然衔接, 形成连贯语流.

## Code Review

Tom: Hey Sophie!

Sophie: Hi! How are you?

Tom: All good, all good. So I've reviewed the changes that you made to the ETL pipeline. Overall, it looks great, but I've got a few points I'd like to **go over**.

Sophie: Sure ok, let me **pull it up**. Ok I'm ready, go ahead.

Tom: Fristly, in the data transformation phase, I noticed a nested loop structure that might impact performance when we go handle large datasets. Have you thought about optimising this bit?

Sophie: Yeah, I see what you mean. So, instead of a loop, what are you thinking?

Tom: I was thinking you could use a list comperhension for that part.

Sophie: Ok sure, let me go back and review it and I'll **give that a go**.

Tom: In terms of error handing, I noticed some areas where execptions aren't being caught properly. These are crucial since it means it's not going to crash the entire system.

Sophie: Ok good point. I'll **have a go at** adding some try-except blocks here and then I'll go over the error logging to make sure we've got details if there are any exceptions.

Tom: Sound good. On a positive note, I really like **how you refactored** the data loading module. It's much cleaner and easier to follow now.

Sophie: Oh yeah it was a bit of a mess to be honest so it did need a good **tidy up**. Any other points?

Tom: Nope I think that's everything, overall, it's **looking really solid**. I'll leave some comments on the code with everything that I've mentioned for improvement, but great work overall. Well done.

Sophie: Perfect. Thank you. Thanks for the feedback. I'll get started on those and then I'll let you know when it's good to go.

Tom: Alright. Thanks! Have a great day. Bye.

Sophie: Yep and you! Bye.

词汇解释

- **go over**: discuss
- **pull it up**: open the code on my scream
- **have/give a go**: try
- **how you reafactored**: the changes you made
- **tidy up**: reorganizing
- **looking really solid**: very well structured
