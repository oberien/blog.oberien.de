---
layout: post
title: "Interruptions cost 23 minutes 15 seconds, right?"
---

You've likely read lots of blog posts stating that it takes 23 minutes and 15 seconds to get back to work after an interruption, context switch, or meeting. Thus, "do you have five minutes" ends up not only costing those few minutes, but instead about half an hour. But where does that number come from?

I just wanted to quickly reference this fact to a colleague. Quick search for the reference, copy'n'paste it, in and out, 20 minutes adventure. I quickly found a reference to a paper. For sanity sake, I just wanted to verify where it states the 23 minutes. Open the paper, Ctrl+F for "23", "no results". Huh?

### Papers

Most of the posts mentioning the number refer to the paper [The Cost of Interrupted Work: More Speed and Stress](https://ics.uci.edu/~gmark/chi08-mark.pdf). The authors performed a study investigating different effects of interruptions during long tasks.  
Contrary to the quoted number, the study found out that the time spent on only the original task was in fact lower when interruptions were present (20.31 and 20.60 min) compared to no interruptions (22.77 min). The paper never goes into details regarding the recovery time between finishing the interruption and getting back to the original task. The paper never mentions the number `23`.

Maybe it's in a different paper? Related Work? References?
* [A Diary Study of Task Switching and Interruptions](http://erichorvitz.com/taskdiary.pdf) let participants record interruption diaries. The paper does not include or mention task switch recovery time. Its primary result is that the average person has 50 task switches per week.
* [Disruption and Recovery of Computing Tasks: Field Study, Analysis, and Directions](https://erichorvitz.com/CHI_2007_Iqbal_Horvitz.pdf) states that it takes 11 - 16 minutes to resolve an interruption until getting back to the original task. Some of that time is spent to get the mind back into the original task. However, no further investigation of the recovery period has been performed.
* [If Not Now, When?: The Effects of Interruption at Different Moments Within Task Execution](https://interruptions.net/literature/Adamczyk-CHI04-p271-adamczyk.pdf) states “An approximate value for Resumption Lag, the time a subject takes to switch focus back to primary task after interruption, was also collected.” However, the paper never provides any value for that number and doesn't discuss it further.
* [No Task Left Behind? Examining the Nature of Fragmented Work](https://ics.uci.edu/~gmark/CHI2005.pdf) focuses on the probability that a task was resumed on the same day in regards to recovery. There is no mention of a specific recovery time.

### Blog Posts

The search continued. In addition to the 5 papers I (fittingly) read through a total of 23 posts.
* 9 posts incorrectly referred to one of the papers; one of them even included a quote that is nowhere to be found within the referenced paper
* 2 posts correctly referred to the first paper for its actual results
* 9 posts directly or indirectly refer to three interviews with Gloria Mark (the author of the original paper), in which she stated the 23 minutes and 15 seconds recovery time
* 2 posts refer to the Wall Street Journal, which directly quotes Gloria Mark with the 23 minutes 15 seconds figure

So in the end, where do the 23 minutes and 15 seconds come from? They are mentioned in interviews multiple times by Gloria Mark. But I wasn't able to find a primary printed source. There are [many more publications by Gloria Mark](https://ics.uci.edu/~gmark/Home_page/Publications.html), but none of them turned up while searching for the 23 minutes 15 seconds figure. If someone knows a paper or study where that figure originally appears in, please tell me.

---

Discussion on [r/programming](https://www.reddit.com/r/programming/comments/17ooxwe/interruptions_cost_23_minutes_15_seconds_right/).

---

Here is the reference graph of all posts and papers I've mentioned in this post and a list of their links.

![](/assets/2023-11-05-references.svg)

* Dev Interrupted -- <https://devinterrupted.substack.com/p/3-proven-ways-to-improve-dev-focus>
* Loom -- <https://www.loom.com/blog/cost-of-context-switching>
* Paladinic -- <https://www.paladininc.com/blog/detail/6299/dealing-with-work-interruptions>
* Lifehacker -- <https://lifehacker.com/how-long-it-takes-to-get-back-on-track-after-a-distract-1720708353>
* The Muse -- <https://www.themuse.com/advice/this-is-nuts-it-takes-nearly-30-minutes-to-refocus-after-you-get-distracted>
* Fast Company -- <https://www.fastcompany.com/944128/worker-interrupted-cost-task-switching>
* idonethis -- <https://blog.idonethis.com/distractions-at-work/>
* idea to value -- <https://www.ideatovalue.com/curi/nickskillicorn/2023/07/it-takes-23-minutes-to-regain-focus-after-a-distraction-task-switching/>
* LeadDev -- <https://leaddev.com/process/managing-chaos-context-switching>
* getabstract -- <https://journal.getabstract.com/en/2022/03/17/twenty-three-minutes/>
* gallup -- <https://news.gallup.com/businessjournal/23146/too-many-interruptions-work.aspx>
* JournalStar -- <https://eu.pjstar.com/story/news/2013/01/25/frequent-emails-phone-call-interruptions/42450766007/>
* togglblog -- <https://toggl.com/blog/how-to-get-back-on-track-when-you-get-distracted-at-work>
* Presentation by Gloria Mark -- <https://slideplayer.com/slide/1409624/>
* Productivityjunkie -- <https://www.linkedin.com/pulse/productivityjunkie-23-minutes-15-seconds-mystery-sugar-inga-bieli%C5%84ska>
* Bright Developers -- <https://www.brightdevelopers.com/the-cost-of-interruption-for-software-developers/>
* Wall Street Journal -- <https://www.wsj.com/articles/SB10001424127887324339204578173252223022388>
* Ironistic -- <https://www.linkedin.com/pulse/cost-distractions-developers-ironistic-com>
* Jazz Hanley -- <https://www.linkedin.com/pulse/reclaim-23-minutes-you-lose-every-time-youre-work-jazz-hanley>
* devmio -- <https://devm.io/careers/aaaand-gone-true-cost-interruptions-128741>
* Stephanie C. Mitchell -- <https://www.stephaniecmitchell.com/articles/whats-distracting-you-from-writing>
* devbizops -- <https://devbizops.medium.com/getting-into-the-developer-flow-state-7b0e5c98eb8a>
* Hardvard Business Review -- <https://hbr.org/2014/04/help-your-employees-find-flow>
