---
title: How we onboard new iOS developers at Revolut
date: 2022-07-12 21:55:04
---

Onboarding can be daunting, especially in a team of 94 iOS developers. When I joined Revolut in March 2017, there were no onboarding. I got started by doing bug fixes. A lot of bug fixes. It was fine, especially for a small company. But what works for couple of iOS developer doesn’t work for nearly one hundred of developers.

Mainly because as an iOS developer at Revolut you mostly work on 4 main areas: view models, repositories, views and flows. And this approach with bug fixes doesn’t work well because first at all you can’t guarantee that a new joiner will get bugs that are evenly distributed across all these 4 areas. Secondly when you’re fixing bugs in an area it doesn’t necessary translates into a skill of creating a new object within this area since you don’t see the big picture. Thirdly bugs are uniq and usually either complicated or trivial. Trivial bugs don’t teach you much and complicated bugs require a lot of expertise so it translates into a lot of supervision thus making this approach not scalable.

When I was talking about this problem to my colleague and he told me that some companies prepare a dummy project for new comers so they can be better prepared. I loved this idea and since at the time my team was getting a new iOS developer and I decided that it was a perfect opportunity to test it out.

But before we go into details I wanted to discuss why I thought it was beneficial to setup a dummy project like this.

## Safety and reduced mental pressure

Since we’re not going to release this code to actual users we don’t risk shipping buggy code. Also it reduces mental pressure on new developers since they know they can make mistakes and it will not affect KPIs or other team members.

## More effective learning

When I designed the dummy project and I knew what I wanted to teach new developers. So the whole thing is designed around this goal. So if you do something you do it to master a specific skill. In contrast to bug fixes when you can’t control what exactly you need to do so you might waste a lot of time doing things that teach you nothing.

## Wider understanding

During onboarding you’ll work on all 4 major areas so by end of the course a new joiner gets a solid foundation. They understand what goes where and they get the big picture. Again in contrast to bug fixes where you can’t guarantee that a new joiner will get tasks in all major areas.

## Better scalability

Since existing developers help new joiners to perform onboarding it’s much easier to describe the one much more predictable process rather than chaotic process of bug fixes. Moreover since we’ve been using this approach for long time, existing developers help new joiner with onboarding while they had exactly the same onboarding experience so they’re on the same page with new joiners.

Now I want to talk more about the onboarding itself. The onboarding itself is a JIRA story that has 7 tasks in it. And every time we need to start onboarding for a new iOS developer we make a copy of the story and assign it a new joiner. But before we start looking into these 7 tasks let’s talk about a buddy. Because in my opinion a buddy is the most important ingredient of a successful onboarding. So once you join Revolut we assign a buddy to you. Your buddy is usually a more experienced iOS developer from your team. And he/she will be there to help you learn. Now let’s talk about the tasks that we have.

## General discussion

The first task is all about talking to your buddy. Here your buddy will give your a bird’s eye view of the whole thing. Including general structure of the app, tooling, git practices, unit testing, view models, repositories, ui testing, CI pipeline, localisation, feature toggles and more. We don’t expect people to grasp all of this but we want to touch as many areas as possible to give you sense of scale.

## Create module

Here we start by creating a brand new module and we use this module to keep all the code that a new joiner is going to write. Since our codebase is highly modularised this step is necessary to tech people how to create new modules and how to integrate them into the main application.

## Create view

In this task you need to create a reusable view by using our design system. Here the view itself is trivial. It consists of a label and a segment control. The view itself is trivial because we want to focus on everything else, mainly structuring code, using parts of design system and writing snapshot tests.

## Create repository

Here we learn the repository pattern. In our case we pretend that we have two endpoint and we need to create a repository with persistence layer. One endpoint is giving you a lottery with bunch of numbers to select from and the second one is used to submit your lucky number. Here the main focus is again around structuring code and unit testing.

## Create screen

Here we combine your work from two previous tasks. You create a screen that should give you ability to pick your lucky number and submit it. So you use the repository to obtain all available numbers than you create UI of the view you created. Once you click a button you submit the selected ticket by using the repository. Here again the main focus is structuring code and writing unit tests.

## Create flow

The last and the most challenging task is to create a flow which should present the screen you created previously and then show success screen (which we provide) after you submitted your choose. Again we focus mainly on structuring code and unit testing.

## Postmortem

This is my favourite. Here a new joiner get a chance to help other new joiners by providing helpful feedback about the onboarding. Because of this step our onboarding is always up to date and we get helpful insights from new joiner’s perspective. But also we get a lot of kind words 💌

As you can see the main theme of this project is trivial objective, compound nature of work and focusing on structuring and testing of the code. You can imagine that we want people to be able to easily understand code of other developers and the onboarding process helps a lot. And these 66 iOS developers that have gone through this process can definitely say it helped a lot.

Worth noting that a new joiner doesn’t take any real production tasks until they complete the onboarding tasks and it usually takes between 1 and 2 weeks for a developer to finish this process.
