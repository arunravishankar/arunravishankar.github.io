---
category: Projects
---

# Data-Driven Safety Countermeasures

## Background
The transportation engineering field has established guidelines for when to apply traffic safety improvements to our streets. In California, the primary document that contains these guidelines is the California Manual on Uniform Traffic Control Devices (also known as CA MUTCD), the latest edition of which can be found [here](http://www.dot.ca.gov/trafficops/camutcd/). They specify the specific circumstances under which traffic control devices should be installed, depending on the goal of the traffic device (improve vehicle flow, safety). For example, to qualify for Traffic Signal Warrant 7, Crash Experience, within CA MUTCD there must be a minimum of five correctable collisions within a recent 12 month period.

LADOT engineers have always used these CA MUTCD guidelines to justify where they should install new signals and protected left turns, and, importantly, these criteria are used to apply for grants for improvements. For example, cities in California routinely apply for funding through the California Highway Safety Improvement Program (HSIP) for infrastructure improvements to improve safety. Under the HSIP guidelines, locations selected for new traffic signals must satisfy Warrant 4, 5, and/or 7 in the CA MUTCD. As such, when searching for qualifying locations, screening criteria must be discretely tied to the requirements of these warrants.

Historically, LADOT staff screened for HSIP candidate projects based on an existing departmental list of authorized improvements, many of which were initiated by public request. There are two potential shortcomings to this approach:
1. Intersections only receive attention when a citizen initiates a request. Areas where citizen engagement is low would be likely missing out on improvements. To some extent this is ameliorated by the excellent local knowledge of our district engineering staff. However, there are still opportunities for high-need intersections to fall through the cracks.
2. Because we would be missing out on addressing high-need locations, our funding package will be less competitive than it othewise would be. This is because a key piece of the application includes calculating a cost/benefit ratio for the improvements.  

## Purpose
The purpose of this project was to find a way to automatically screen intersections based on these same criteria we have always used to apply for funding. We wanted to look at every intersection in the City and then check the criteria to see if it would qualify for an improvement and caluclate the cost/benefit score. This would address the two shortcomings above in the following ways:
1. Because the process would be data-driven, no intersection would be passed over (unless there was an error in the data), ensuring that every area was treated fairly.
2. We could put together the most competitive package available.

## General Approach
Although the theory behind the approach is simple (counting the occurrences of specific collision types), the implemention required a substantial amount of data prep work. First, we assigned every collision to an intersection. For each intersection, we then looped through all the collisions assigned to it, checking for specific criteria. If the collision met the criteria, we kept it, and if not, discarded it. Finally, we checked each intersection's resulting collision history to see if it had 5 or more qualifying collisions within a recent 12 month period. Once we had this list, we could then rank them based on the expected cost/benefit ratio calculator provided by HSIP. Pretty straightforward.

See below for a flowchart diagram of the specific implemention for the identification of new signals.

![](/images/2017-01-10-DataDrivenFunding/HSIP_CityWide_SignalWarrant_portrait.png?raw=TRUE)

## Outcomes
We used this process to apply for two types of improvements: new signals (using the process shown above) and protected left-turn upgrades. We were able to apply this process for both the HSIP process described earlier, as well as the Metro ExpressLanes Grant Application program. A more formal summary of our process can be found via a TRB paper I co-authored [here](https://black-tea.github.io/documents/TRB2017_VisionZeroBeyond_Paper.pdf).

## Shortcomings
This process only addresses one tiny piece of the safety equation. Vision Zero is really about looking at a systems approach to safety, whereby changes at any point along the network will affect the rest of the network. For this project, we didn't address how these changes would affect the rest of the system (in terms of both safety and operation). We are really only addressing [Strategy 1](https://black-tea.github.io/blog/2017/01/13/The-Engineering-Bucket.html) of the Vision Zero Engineering Bucket -- changing the likelihood of a collision -- instead of also addressing the likelihood of an injury given that a collision occurred.

## Code and Project Materials
The code and project materials can be found on [GitHub](https://github.com/black-tea/VisionZero).
