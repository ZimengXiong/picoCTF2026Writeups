# AI Usage

I used a lot of AI, a lot of it...

## The General Pipeline
Every problem was initially sent to be triaged by an agent, which also set up a testing environment for it. Depending on the problem, the agents would either: solve it immediately (if easy), triage and propose solve paths, ask me for manual assistance, or completely fail.

## Handling the Easy & Mediums
For the easy problems (that I knew exactly how to solve by looking at it), I just dicated to the agent:
> "find the flag by `solve`."

### Triaging medium problems
If it was a problem that wasn't super obvious AND was not HARD difficultly, I would open the harness session in a new tab and send the agent off to explore it.

### Starting the hard problems
That meant 80% of the problems were now being looked at by the agents. In that timeframe, I started focusing my attention on the hardest (500pt) questions. I didn't use my harness for these hard problems (because I expected a lot of back and forth w the the agents was required for the them to solve the questions), so I just opened many sessions manually and sent the agents off to try their best at triaging the solution path for these hard problems.

### Reviewing medium solutions
By this time, the medium problem solutions paths were coming in. A fair percentage (around 30%) of the agents requested assistance. 

The remaining 70% had clear solve paths. PER THE CONTEST RULES 😜🙂, I went and read over AND understood the solutions for these solve paths (I acc did ts), and in a NEW worktree I dictated to a context-fresh agent the solve path as I understood it, and sent it off to produce the flag. 

#### Working with the stuck agents
For that 30%, I started to work with them on the problems. 

A lot of issues were due to the the agents not being able to OCR text correctly (I specifically stated in my  agents.md to NEVER attempt to use OCR to produce flags, and always ask for help when they are given an image with the flag)

Some others complained about not having access to an x86 system as I used a Macbook for this (as I again, specifically refused Docker emulation in my system prompt). 

There were really only 6 ish problems where the agents were genuinely stuck in this medium category of questions, which I worked through with them more in depth and solved.

### Hard problems
Now for the fun part! All easy+medium problems had been solved by this point. I found the agents generally able to solve most problems, except for `Pizza Router`, `paper-2`, `JITFP`, `Secure Dot Product`, and `Binary Instrumentation 4`. I used cmux to organize all my agent sessions by difficulty of the problems. 

Around 50% of the hard problems required my guidance in a couple of follow up prompts and brief discussion about where the exploit really was, 25% were able to be full solved by the agent alone (in the same method I described in [Reviewing medium solutions](#reviewing-medium-solutions)), and the remaining 25% of the agents were completely stuck. 

After resolving the agents which could be resolved with assistance, we were left with those 6 problems. Here, we pulled out a secret weapon: `/model gpt-5.4-medium-codex`. As we will explain later, we had been using `gpt-5.1-mini-codex` thus far (scary, i know 👻). After some collaborative triaging with `5.4-medium`, we solved `Pizza Router`, `JITFP`, and `Binary Instrumentation 4`. 

Now only `paper-2` and `Secure Dot Product` were left. Codex was fully convinced that `paper-2` required a JS exec exploit, and I (at the time) thought the same, so I let it try it's best at that while I manually solved `Secure Dot Product` (in that I was only asking codex to write code, I forget what specifically but it was stubbornly insisting on some obviously dumb exploit)

Paper 2 took some manual research and asking different agents more creative than codex (Gemini) on other possible attack vectors on different components, which is what eventually lead to the identification of [redis key eviction](https://redis.io/docs/latest/develop/reference/eviction/) as a possible exploit path.

## Technicals

### Agentic Agents
I wanted to see how far the normal person, spending 0$ on agents, could get. I only used the free tier of services (Codex, Gemini), so I tried to conserve usage by using worse models (5.1-mini), so when I specify "agents" or "the agent" above, I am specifically referencing my experience with 5.1-mini. Better agents would likely have produced better results.

### Harness
I built a problem intake/solving harness. For each problem, it would:

1. Spin up a `tmux` session for that problem.
2. Launch an interactive codex session within
3. Feed a custom prompt file with instructions based on the type of problem (designated at problem intaking), alongside a custom CTF-skill I wrote specifically for jeopardy-style CTFs (triaging methods/common solutions by problem category)

The harness used a web UI, and would stream the `tmux` session so I could interactively communicate with sessions in browser. Agents could request help/assistance and be surfaced in the UI. Agents could also declare a `flag` for review. Agents who submitted a flag for review or help/assistance request would trigger a new tab to be automatically opened with that agent's interactive session. The median time to flag was around 10 minutes for the 60 problems entered into the harness (the hard ones were not placed into the harness, because there was a lot of back and forth and manual intervention that would not be possible in the harness).

### Misc
I built a chrome extension to make it easy to copy the problem statement, all links/artifacts, and hints.

I did not use browser automations to launch instances/submit flags. I manually went over and started all instances, read problem statements, and submitted flags.

I prioritized non-instance problems and problems with remote source code given, as the #1 error agents reported was that the instance was unreachable.

## Reflections

### About picoCTF
I don't see myself doing anything different next year given that AI were still to be allowed in the competition (besides using better agents, this time I really just wanted to see how far the normal person could get for free, which went pretty well I would say). Using AI brings a different kind of thrill (in building agentic systems) rather than exactly solving CTFs.

I definitely did not have as much involvement in the problem solving process (really only around 10 problems were over 80% authored/written by me out of a total of nearly 70).

I think in the future, we could maybe have an AI and a non-AI division, or atleast a no-parallel-agents division? I think that is easier to enforce/spot than prohibiting AI outright. Or we could take the path to make problems harder (UnforgottenBits, Ricochet, Virtual Machine 1 come to mind). More visual problems, puzzles, especially considering introducing OSINT categories??, etc could be a way to combat AI one shotting all problems while still keeping picoCTF at an introductory level.

### About agents
I have to say Codex is (generally, in my experience) one of the least creative agents. After I got `paper-2`, I wanted to see how fast other agents could solve it. GPT 5.4 Pro never produced a correct solution (nor any of the GPT family models). The same with Opus 4.6. 

#### Low Cortisol Gemini 
Only Gemini 3.1 Pro was able to one-shot a solution to `paper-2` when prompted 3 times independently, all of which produced different results, take that Codex! (Gemini 3 Flash was not able to do so, it kept insisting on a JS exec bypass via PDF, also worthy to note that I did not have access to Gemini 3.1 Pro on the free tier, so I did not ask it during the challenge). 

More impressively, Gemini has the weakest harness (limited to Python), while 5.4 Pro and Opus 4.6 both had a minimal linux shell and node enviorment (though docker was not avaliable in either), this means 3.1 Pro solved the problem only through static analysis, and without any scripting. Had 5.4 Pro had docker access in it's container or some way to experiment, it may have been able to produce a solution, sadly there is no codex (the CLI) access to 5.4 pro, so we'll never know (unless someone wants to foot the API bill).

#### Gemini 3.1 Pro
![](gemini-solving-oneshot.png)

#### ChatGPT 5.4 Pro Extended Thinking
![](chatgpt-pro-failing.png)
