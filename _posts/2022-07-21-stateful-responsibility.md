---
title: "Stateful responsibility"
date: 2022-05-31T13:35:53+02:00
categories:
  - frontend
tags:
  - frontend
  - vue
  - vuex
  - state
  - state management
toc: true
---

When the work on Elixeum portal began, every **Vue** component used to handle its own requests by itself, that ususally meant that every list component had locally implemented API call to count, to list with paging options and at least to delete request for specific element of said list. And every detail component had all of the CRUD API calls integrated into it as well.

**This quickly proved to be unsustainable** as large chunks of code were being copy/pasted across many components, sometimes even without changing a single bit of code - and that's without even mentioning that we had to pass all the data across components using props so that we can at least partially minimise amount of load being pushed against the server.

And it got even worse when first change request from business side came through - now we had to go through all this hell and update it everywhere.

We obviously needed something better, something more robust, something that's less PITA and more enjoyable to work with.

# States provide a new hope

Vue obviously provides solutions for all of the above mentioned problems, no matter if you choose to use **Pinia**, **Vuex**, or any other state management library, you will get rid of the headaches due to clear separation of responsibilities.

![State management with Vuex](https://vuex.vuejs.org/vuex.png "Thanks Vuex team for providing this beautiful diagram!")

As seen in the diagram above, there are is clear order and definition of which part of your codebase is responsible for managing API connection, which one stores the data, which one provides it to the UI and how the UI communicates back to the backend.

Not to mention that this also allows you to remove all the duplicated code from before, store it in one place and just use it when needed in any UI component. Great start!

# Copy/paste strikes back

But then again, all of your state files will usually have the same structure:

- They will likely import some kind of Consts file with API endpoints
- They will likely handle your general data management problems such as paging or filtering
- They will likely perform all CRUD operations in addition to listing and counting

And then in some cases, they will perhaps perform something more specific, but most of the time it's at the very least the same thing, just over different endpoints and different datasets. And the lure of the "copy/paste" dark side of many developers starts to get strong again...

![Copy/paste dark side](https://c.tenor.com/HXkIETckaikAAAAC/kermit-darkside.gif "Dark side of the Force will lead you to many copy/pastes")

What would a Jedi do? Well, apart from destroying the Death Star again... they would try to figure out how to create a template of sorts for the states that share a lot of the same logic. And they would think to themselves - oh look, some of our shared codebases come from modularized npm packages, let's put it in there and implement the package across the whole codebase, that's going to be great! Even if it comes at the cost of slowly migrating most of the already existing state logic to this templated code.

> At the end of day, every UI modules registers it's own state files, they can be named anything the developer might find fitting, so why not go this route?

All you have to do is to pass the templated code metadata that identifies endpoints to call, how the state should be registered and named - and if there is anything extra, what's to stop you from just using Javascript's fancy spread operator to extend an existing state (at the end of the day, it's just an object) with your custom logic? That's right - nothing.

# Return of the responsibility

It seems like everything about states is already figured out, it's working, it's kind of magical and it's saving us headaches... well, sometimes. It's not completely related to state management as much as it is related to how you actually work with data across **Vue** components...

You see, for some time it seemed like a great idea to just have every component bind to a specific state and to some extent abandon the way of passing data through props altogether. That works great - if all your components are single purpose and specifically tailored for the use case they had prescribed in their initial UserStory ticket. But all it takes is one change request, or one user story - that contains the magical word "repurpose", or "reuse", or "use existing"... you get the idea.

You seemingly have 90% of your work already done on UI side, all it takes is an _if_ here and _if_ there - and bam, in 2 hours you're done... until that component requires you to use a different endpoint with some difference in data structure - and it breaks apart completely.

# Summary
