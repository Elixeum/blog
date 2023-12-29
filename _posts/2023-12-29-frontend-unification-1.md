# Frontend unification (Pt. 1)

First part of shorter series of blogposts about major refactoring of our frontend technological stack to solution that seems very obvious now, but really didn't three years ago. Going by memories and rather big pile of notes we wrote down while working on this fun endeavour...

In some parts we shall go more in depth with technicalities, in some parts we'll focus more on the "story" and challenges that emerged.

## How did we get here?

Nearly three years ago at this point in time, foundations of Elixeum portal were being made. These foundations were built on certain business core ideas, such as having clear and fairly "definitive" separation between each micro-service, on both backend and frontend level.

It's fairly straightforward concept in terms of how to implement it on backend, especially with .NET codebase, however how do you neatly tie that in with frontend codebase as well - provided that you don't intend to use .NET's native frameworks, such as Blazor. Which we didn't.

No, our web frontend tech stack is built on Vue 3 (back then still in beta stage, by the way), so in order to achieve the same level of separation on frontend level, the simplest possible solution that came to mind to us was to just create a baseline Razor page, which would only do one thing - load up full fat `index.js` entry point file of the actual Vue application.

![So that was a f... lie](https://i.imgflip.com/8arhqd.jpg)

So yeah, the first paragraph about not using .NET native frontend frameworks was a lie, from certain point of view. Most important thing is, this solution was sufficient in the beginning, and actually worked fairly well once we got around some issues with build hashing and serving correct fallbacks.

In addition to this entry point service, we had a system in place that allowed other service to connect into it - when you made a call to endpoint on specific micro-service, you would get back response with path to it's service entry point `index.js` file, which would lazy-load in the frontend for it into an already existing application served from the entry point micro-service.

## Works well, but…

As you can probably imagine, there are some inherent issues with this approach to code separation, and those only got bigger as time went on, the list of issues includes:

- Issues with hashing.
- Problematic obfuscation, since every frontend was being built as a complete separate entity.
- Problems with sharing of components and logic. Worked around it using shared logic npm packages, but that also lead to a lot of micro-management of dependencies later down the line. Not to mention that locally developing inter-dependent packages in final project is also about as pleasing as eating concrete.
- Working with localisations that were shared across micro-services was cumbersome and basically lead to a lot of copy-paste.
- Practically impossible to load balance frontend serving.
- Making solution-wide changes to frontend was an absolute PITA, just updating all dependencies correctly took a full day for developer who knew what they were doing, not to mention that dependency watchers reported many duplicates, since there were multiple separate builds present
- Why even serve frontend app from backend server, if said app isn't built using server-side-rendering?
- Frontend-only developer really doesn't need to go through the entire process of setting up backend, databases, building entire solution, and running it on their local machine, provided that development server runs it just fine - all they need for their work is backend that serves data to the frontend. But you can't really get there when it's backend that serves the frontend...

Many of these issues have workarounds, they are not pretty, but they work… for some time… but to add some extra salt to injury, the business core ideas shifted as well, so there was a big push towards interconnection of each micro-service and sub-system on both backend and frontend level. It was clear that foundations need to be rebuilt, to certain extent.

![This is where the fun begins](https://i.imgflip.com/1ka4ul.jpg)

## This is where the fun begins

Now, since we are actively intending to rework a lot of codebase with the sole intent to actually make it a lot better and easier to work with, why not look at our fundamentals from the ground up. Every core micro-service frontend was based on Webpack bundler, but since Vite was getting a lot of traction, we experimented with it a bit in our npm packages - and with very pleasing results, such as vastly lessened build times and shockingly good ease of setup compared to Webpack even with simplest things such as SCSS pre-processors. Since this entire endeavour was planned for roughly 3 months of work (in reality more like 4 and a half), what is a day or two extra for some update to new bundler… well at least we thought that it would be just two extra days. This entire process was also connected to us moving on from Yarn Classic 1.x to modern Yarn - at the time version 3.x, by the time we finished it was already on 4.x though. We'll get to all that later.

And you know, while we're at it, let's rework how we work with localisation files - nothing is more fun than merging 10 csv files into one and fixing duplicates that even vary based on casing - and we are not talking about duplicates in keys. Those are the easy part. But to unify localisations across multiple languages… yeah, great fun. Let's add that in as well!

Final cherry on top wasn't really apparent on day 1, we initially planned to trim down our npm package logic separation quite a lot later down the line, but it became very apparent that it needed to be done as part of this entire rework, because most packages needed to be modified as well, due to core project having some required changes in order to support connection to API on different URL than the one from which is the frontend served. So we had a choice in front of us - either we maintain over 10 development branches of those packages, or we just say "damn it" (we actually said it the less polite way, obviously) - let's just do it now as well and unify previously packaged logic into core project right now.

All that while existing work of other members of the team must not be blocked in any ways, so yeah, in the end, we will do a lot of re-merging, before making the final switch.

![Captain's log](https://i.imgur.com/x0wAW.jpeg)

## Captain's log, day 1

First day went actually really well. There were more positive notes than negative ones, to say the least. However it became already apparent that we would need to something about our npm package situation sooner than expected, even though it was planned for later. And yes - let's admit it, we were a bit scared of all the things that could break and go wrong.

### The good

1. **Basis is there:**
   We kicked off our new project with the combination of modern Yarn and Vite, laying down the more modern basis than previously used Webpack + Yarn Classic combination. For now still running on the old repo, just separate folder - in order to preserve ease of use. This will change later down the line.

2. **Kinda-sorta-copy-paste done:**
   Migrating the existing project code seamlessly into new folder marked a strategic move, bringing coherence and organisation to our codebase.

3. **Build triumphs:**
   With some caveats (see in "The bad"), we successfully got the build working, setting the stage for a smooth development process.

4. **Styling simplicity:**
   Basic styling worked out of the box without any extra setup, something we weren't used to when using our SCSS + Vue Scoped styling with Webpack.

5. **Unprecedented speed of build:**
   The production build time witnessed a staggering improvement, dropping from roughly 36 seconds to an astonishing 8 seconds. That was a mighty morale boost for further work. And yes, this was tested on equal settings regarding minification and obfuscation - with both disabled. With enabled, Vite build is obviously higher, but still keeps same relative gap to Webpack, which also slows down a lot with said features enabled.

6. **Reload magic:**
   Hot reload functionality worked seamlessly, adding a touch of magic to our development process. Something that many take for granted, but our previous technological solution sadly made it borderline impossible to achieve in any way that would actually improve our productivity.

7. **What about the environemnt:**
   Vite provided solid basis of build modes, which allowed us to easily separate target environments from localhost to develop, alpha and main. These were seamlessly integrated, enhancing flexibility in our development pipeline and allowing us to run local frontend build against any (provided that CORS allows it, obviously) server. It wasn't ideal solution as we had imagined it, but it worked well for first day attempt.

8. **Vue voyage:**
   Successfully navigating from Tenant domain entry to Login screen and executing a flawless API call to `app-config` showcased that we are on the right track. The entire project became a source of joy, because things were going better than expected… at least for a start.

### The bad

1. **GRPC puzzle:**
   The road ahead includes reworking GRPC due to pregenerated protobuf files incompatibility with Vite. A mystery that demands our investigative prowess - at that point we had no idea what to do about it, but - spoiler alert - we figured it out. This was also pretty "fun" experience provided that other part of the team was actually in the process of implementing GRPC into old solution, so we already knew at that point, that it could be subject to change, but still had to think about solving it.

2. **External quirks:**
   Importing issues surfaced with libraries like `PdfVuer`, pointing to package-side challenges - mostly just due to direct incompatibility with Vite build stack. Solutions seem rather straightforward, but only on the surface - either find an alternative on GitHub, or write your own.

3. **Import oddities:**
   Some libraries required an unconventional import syntax - `import * as name from 'package'` instead of importing default export from a package, seems weird.

4. **Build mode compromise:**
   Custom arguments for build configurations were desired, but Vite's limitations led us to embrace build modes. Another spoiler alert though - we figured that out later down the line as well, and it was actually pretty straightforward.

5. **Styling conundrum:**
   Despite a successful API call to `app-config`, challenges arose in implementing tenant-based custom styling for the login page, warranting a deeper investigation once more important issues are resolved.

## Captain's log, day 2

### The good

1. **Localised success:**
   Localisations are now seamlessly bundled without the need for pesky API requests that provided them previously across all micro-services. A significant stride towards a more efficient and self-contained system.

2. **Login is working:**
   Successfully managed to log into the app just like in old solution. It's apparent now that there will be a need to update all API calls across entire application though, as it previously relied on having same host for API and frontend.

3. **Visual appeal:**
   The implementation of a loading breathing logo while resolving background tasks has been accomplished effortlessly, adding a touch of visual finesse to the development environment. Just added an image directly to base HTML file that is present until JS app is loaded - and then the APP "hard-replaces" the image with its own content. Simple, yet elegant for the user experience.

### The bad

1. **Port woes:**
   Despite efforts, achieving a "portless" Vite serve for development purposes proved complicated. The port will persist in the URL. However this is just a minor cosmetic issue for developers, and has no effect on actual usage.

2. **Localisation headaches:**
   Extending localisations across previously "micro-serviced" frontends poses a challenge, demanding boring work to be done on unification of keys, as previously they were completely independent of each other, but now are all present in one source and we need them to not conflict each other.

3. **CORS, of course:**
   The CORS policy emerges as a roadblock, preventing the local web instance from running against the server. A specific error highlights the need for addressing `Access-Control-Allow-Origin` headers. As usual with spoilers - we resolved this, and in the end just meant we had to allow all origins on development builds for our server environment, while keeping them tight closed for production server environment.

## Captain's log, day 3

Overall this day seems a bit less stacked with updates, however there are some key changes that were thought through and planned for upcoming days.
These changes mostly revolve around our service identification systems, since API still remains divided in micro-services, we needed correct way of pointing which service said component is related to - even if it's available at any point, based on licensing of our customers (tenants), we might allow them to use only a specific set of components, as well as maintaining which API should states of said components call.

1. **Update service references:**
   All instances of `Service.ServiceName` will be replaced with `Service.Name` - why was there a duplicate Service keyword in the name of the property? That's a mystery for another time.

2. **LocalisationKey cleanup:**
   Since all components are now sharing localisation keys across entire codebase, there is no need to specify `LocalisationKey` as separate identifier for localisation. You might ask why was there a separate key for `LocalisationKey` from `ServiceName`… and you would ask a good question. Yet another mystery though. Yes, we are doing a bit of self-roasting here. 🔥

3. **State path adjustments:**
   A careful update of all state paths is required to align with the revised naming conventions. This is the most painful one to execute, as it's the least possible one to just handle using "replace-all" method. Some paths are based on ServiceName, some are hard-coded, some are based on specific sub-functionality… yes, it means manually going through entire codebase and fixing… as well as just reading browser console and detecting issues. Fun times.

   ![Replace all](https://www.monkeyuser.com/assets/images/2018/115-replace-all.png)

### The good

1. **Triumphal victory over CORS policies:**
   After some adjustments to the backend related to CORS policies - namely allowing correctly all origins on specific environments (non-production ones), the much welcome achievement of running the local UI against the server API has been unlocked. A pivotal moment that opens up new possibilities and streamlines the development process, especially for frontend-only developers.

### The bad

1. **Lazy loading issues:**
   The current hurdle revolves around the goal to lazy load an entire service. It's obvious that this issue has to be resolved using dynamic imports, as we demonstrated that on small proof-of-concept project, however applying this on larger scale is not as straightforward as expected. There are issues with missing Vue components, missing imports of style files and such...

## Captain's log, day 4

The transition to modern Yarn has brought shifts in package linking behaviour that developers should be aware of. In original codebase we used package linking quite a lot for streamlining local development of npm packages and debugging them on locally built frontend. And since we are yet to unify packages with core project at this point, we wanted to use linking again as a development workaround… so overall this day was pretty pointless in the grand scheme of things, as everything from it was scratched later on… but sometimes you need those days to actually get there.

### The good

1. **Streamlined package linking:**
   Unlike in Yarn versions prior to 2, there's no longer a need to use Yarn link in the source package. Instead, linking is simplified in the target project using the following command, which only relies on path:

`yarn link ../../BuildingBlocks/Frontend/ui-state`

This streamlined process enhances clarity and ease of use. However it is important to be aware of this, as attempting to link the package with anything but its path will lead to issues. But yes, as we have repeatedly stated, this will not be necessary at all once we are done with this entire endeavour.

2. **Modern Yarn compatibility with peer dependencies:**
   Yarn 2's enhanced compatibility with peer dependencies is a notable positive point. This improvement reduces reliance on tools like `yalc`, potentially streamlining the development workflow.

3. **First micro-service frontend moved and working 🎉:**
   This is an important stepping stone forward, as we now have a fully working proof of concept for moved micro-service code into unified core project, with properly working lazy loading and base logic.

## Captain's log, day 5

### The good

1. **Component naming cleanup:**
   Amidst the challenges, a positive outcome emerged with a clean-up of component naming. This step enhances code readability and maintainability. We especially had a big problem with a very strange fact, that way too many component shared a `Dashboard` key in their name, even though their functionality didn't even related to dashboards as we understand them at all. This issues was fixed and now only components that truly relate to dashboards have the name in them.

2. **Lazy loading breakthrough:**
   A notable triumph comes in the form of successfully implementing lazy loading for the Calendar component and its states. This change not only optimises the initial load by saving up to 400kB but also sets the stage for a more efficient and responsive application. This is all possible thanks to Vue router's hooks that allow us to load heavy states and components only at the point when they are actually needed for usage.

### The bad

1. **Image encoding woes:**
   In the realm of production builds, a notable challenge surfaced with image encoding after switching to Vite. Images were compiled into Base64 strings, rendering previous checks reliant on file extensions completely ineffective. This necessitated the creation of a new ExtendedImage component to make informed rendering decisions.

   This is due to the fact that some of our images are rendered as FontAwesome icons using `fort-awesome` package, which are only defined as string names of said target icons, our own custom svg icons, which require custom component in order to support styling them properly… and also good old classic images, both in link form and in a form of precompiled Base64 strings, as provided by Vue.

![Is this Base64?](https://i.redd.it/apab79o06z011.jpg "No, it's not")

## Captain's log, days 6, 7, 8 and 9

### The bad

1. **Back to issues with npm packages:** It is becoming more and more clear that we need to get rid of packaged logic, since it serves no purpose in unified codebase, and only causes issues with managing the codebase, slows down the development process and also causes issues - since other parts of the development team continued to work on old codebase, meaning we couldn't just make breaking changes into already used repository of packages. For now a hard copy into separate folder works, but we need to move on soon…

2. **Watch out for Yarn version when switching between parts of codebase:** Apparently when you are inside a project structure, the version changes only for the project, not globally. This led to a lot of pain with attempting to link a package between Yarn classic and latest modern version.

## To be continued

Obviously as work progressed, we had less and less notes for each day, so in further blogposts we will go more in depth with specific issues and how we tackled them, rather than doing this "Captain's log" timeline approach.

If you want a little taste of what is to come:

1. How we cooperated with backend development team on some required refactors
2. How we actually tackled the process of migrating npm packages directly into core project
3. How we tackled actually pretty rough issue with gRPC protobufs not building in the app due to Vite not supporting CommonJS out of the box, and the plugin meant for it only worked for external packages
4. How Vue's `keep-alive` functionality bamboozled us by breaking due to comments in the code, yes - comments!
5. How we went around merging entire updated old codebase into by-then 3 months old base of the new codebase
6. How Markdown editor component completely broke down, but we only found out about it when we were already preparing for production deployment - spoiler, we had to rewrite the entire thing, fun times!
7. How a simple `BOM` in encoding of third party package can make functionality go haywire!
8. How this allowed us to beautifully integrate our global dashboard components that now allow to access widgets from all micro-services, provided that tenants have correct licenses and users correct permissions

Stay tuned for more!

![To be continued](https://i.ytimg.com/vi/z7jkp40ydEI/maxresdefault.jpg)
