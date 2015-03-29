---
layout: post
title: "Developers debate unimportant things"
date: "2015-03-29 14:56"
author:
categories: [software]
comments: true
published: false
---

The other day I read a disappointing screed by a determined developer. The specific text is unimportant, but I'm sure you've read plenty just like it: claims about technology X that the author doesn't like, while simultaneously promoting technology Y that they do like as a technological panacea. Now you know why the specific article is irrelevant -- it could be Sass vs Less, or CoffeeScript vs Dart, or Maven vs Gradle, or K&R vs BSD curly brace placement, and any of those hypothetical articles could be extracted from another with find and replace. 

Consider the case of a hypothetical team with a background in technology *X* that starts a new project in (or migrates to) technology *Y*. The new work goes well, and the team is enthused about *Y*. The question is, though, how important is *Y* specifically to the success of the project? Sometimes, of course, there are cases where technology choice can be a huge factor. Trying to write firmware for an embedded device with an 8K ROM in C++ template metaprogramming (it's Turing-complete, so why not?) is probably not going to go well. I'll set aside these hyperbolic cases, though, as they're rare and easily avoided with a little googling. Let's go back to our hypothetical team and consider the contributions of the following aspects of their successful project:

- Choosing *Y* instead of sticking with *X* or choosing another option *Z*
- The architectural cleanliness that comes from an at least partially blank slate
- Having team members who care about improvement
- Having team members who know that improvement is possible
- Engineering leadership's trust in the team's decisions on what to do
- The organizational health needed to work around a departure from the business-as-usual schedule
- The corporate political bonhomie needed to allow the engineering organization to innovate rather than take the safe road of continuing with what they already have

I assert that in the common case, the success or failure of a project generally has little to do with the specific technology used. In one way, this is not a radical assertion: the importance of organizational structure, team culture, and other such intangibles has been known for decades. (This is what management is all about, after all.) My point is that even though technology usually isn't the important part, *developers argue as if it is*. Developers are generally detail oriented, and many (though not all) are also equipped with a surfeit of opinions. Consider topics like compile time vs runtime type checking, Java vs Scala, tabs vs spaces... We can quantify these topics, so we can bring facts (or at least aesthetics) to bear as we bicker over relative merits. My own experience is, of course, only anecdotal evidence, but across the projects I've been involved in both as a regular employee and as a consultant, the choice of technology hasn't been nearly as important to overall productivity as the team's engineering maturity, organizational health, etc. In conversation with fellow technologists, I've found I'm not the only one.
 
<img src="${r '/images/developers-debate-unimportant-things.png'}"/>

 It's understandable that we might tend to attach too much importance to the things that we know well and have some control over, so where do we go from here? I definitely don't want to write off the value of a good ol' my-type-inference-is-better-than-your-type-inference debate; we all learn a lot from the exchange of ideas. Instead, I think that when we debate such things we have to keep in mind that it's only a small part of success, and that ultimately the specific technology probably doesn't matter much. If you have an opportunity to adopt your technology-du-jour because your team has the political will and organizational freedom to do so, you will have necessarily already succeeded at the hard part: being part of an organization that allows for success in the first place. On the other hand, if you're struggling to get momentum adopting a technology that you're confident will help your project, consider that the tech isn't your problem: it's that you're the only one clamoring for improvement.
