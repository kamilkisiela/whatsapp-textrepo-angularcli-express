# Step 1: Introduction

[//]: # (head-end)


# Step 1: How to build an app?

So you want to build an app!
The good news is, it’s not that difficult and after a while anyone can do it!
Another good news is that it’s a great skill to have, there is a lot of demand in the job market and you can get to be very creative. Also you can be part of basically any industry you can think of if you know how to program (Healthcare, 
banking, education, defense, gaming, etc…).

Software builds as layers on top of layers.  Every code you write is using code that others have written before already and you can just use it even without fully understanding it.
That’s what makes software advance so fast and makes the pace of progress increase as time goes by.
Also, most software today is written in an open and free way - everyone can share their code and use other people's code for free.  We even have a social network for that!  It’s called Github.

One way of learning how to build an app is to study software from the origins. How computers, operating systems and compilers work. It is very interesting but that also takes a long time.
Another way to learn how to create a new app is to start by first using all the code that has been already written out there and once we have a real world running app, go deeper to understand how it works from the inside.

I’ll try to use the later approach because I think it is more fun and also it gives context when we will dive inside on how things work. Also it helps when we try to compare between different technologies that do the same thing (Angular vs. 
React , etc…).
That’s because understanding the bigger picture is more important than knowing and remembering the details.

You noticed that the chapter started with a simple question.
All the lessons will start and include many of those questions.  Those are the questions you will need to Google.  The reason is that the most important skill a creator has today is to know how to search Google for the things they need. Most 
programmers, even the best ones, don’t remember anything, they just use Google for almost every line of code they write. So know that you don’t need to know it all, Google knows it all for you.
This tutorial might get outdated (it won't, hopefully!), but Google will always have the answers so it is more important to know to ask the right questions.
In my opinion the most important skill a programmer needs to have is patience and acceptance when something just doesn’t work, enjoy that feeling and start Googling around in order to learn new things and solve this new problem (this process of 
feeling like you don’t know anything will happen 1000 times a day, also after 20 years of programming).

## Planning the app: how to design an app?

The first and most important phase in the process is the beginning - designing the app, its parts, components and architecture, and how it all works together.
For this tutorial we will build a WhatsApp clone so we have the designs already prepared.

The process will looks like that:

1. **Visually sketch the screens of the app**. We already know how WhatsApp looks so we’ll just copy it on pen and paper.
2. **Break down to components**. The best way to describe the UI of apps is to break the UI into separate components and mix them up together. When breaking down to components we would need to think about the following steps:
  * **Presentational components**: the small building blocks of the UI. For example input fields, images, text, buttons etc…
  * **Layout**: how the components are positioned inside each other (talk about flex, grid and older layout types)
  * **Styles**: colors, fonts etc… We might skip that part at the beginning because we can have a perfectly working app without it
  * **Data dependencies**: the data that those components need - chats, messages, people etc… We will need to to do the following:
    * Gather data for each component
    * Attach all those dependencies into one schema
    * Attach actions for each component
    * Data updates: decide when the data should be updated in the view (ideally we shouldn’t have to say when but for technical and performance reasons it helps)
3. **Routing flow**: Moving between screens, what are the different paths the user can navigate through our app

Once we will finish that process, you will have a big picture in your head (and on your paper) and implementing it with code will be much easier and more technical, meaning you won’t need to know a lot, just Google how to do each step...

Another interesting thing is, that implementing this same process can be done with a variety of different technologies, so this work will make sense whether you are using web with React or Angular, native apps, Windows UWP or any other 
technology for the UI and also relevant whether your data source is a Node, .NET or Ruby server with Mongo, Postgres or MySQL database (if all those words doesn’t mean anything for you, don’t worry, those are just different names to very similar 
ideas and we will learn all of them later on and of course you can always just Google them).

Head over to the next chapter to start planning.


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Intro](../../../README.md) | [Next Step >](step2.md) |
|:--------------------------------|--------------------------------:|

[}]: #
