# Bad decisions

I keep hearing from people about how I should not read an agent's code, and instead just focus on testing the interface. This hasn't worked for me personally. In this repo, I'm keeping track of concrete examples where agents have failed.

## 11/13/26

Part of a general pattern: Whenever Claude needs to fix an issue with one of the tools in the agency agent, sometimes it decides to make the change in the agency runtime itself, instead of fixing it in the tool, leading to a lot of special case logic that match only that specific tool. So in the future, if we have tools that do the same thing, but have a different name or shape, they wouldn't get the fix, and in the meantime, tools that DO have the same name or shape, but do different things, might still get the "fix" applied to them accidentally. eg this pr, first commit: https://github.com/egonSchiele/agency-lang/pull/1458/

Fable 5.2 / Claude Code

## 11/9/26

I had a Fable interaction today that nicely shows the kinds of issues I still have with models. It delves a little into the internals of the agency-lang package, but then I think all interesting interactions do. The issue with explaining when models are bad at coding is, they're not bad at the one-shot examples everyone loves to pull out where an agent does a brilliant job designing and building a website from scratch. It's the cases where they are trying to make edits to an existing project that is very complex. That is where coding agents fail.

Lets get into it. Agency has a built-in agent. This agent has a top-level coordinator agent, and then a number of subagents that it can hand work off to. Recently, I've been playing with using a decision model like Jev to decide which subagents to hand work off to. Jev is orders of magnitude faster than the traditional LLM would be, so its a nice easy way to speed up the Agency agent.
Fable worked through the problem with me. It involved adding support for decision models to another package that agency depends on, and then modifying the agency agent code. It wrote out the plans for modifying this other package. From there, the agency change was simple:

1. modify the built-in `llm` function to support `type: "decision"` + a couple other simple changes
2. Add a Jev call in the agency agent, where we ask it which sub-agent we should route to. Then use a simple match block like so"

```
  match (route) {
    "research" => researchAgent(text)
    "code" => codeAgent(text)
    _ => llm(prompt, mainOptions(turnStart))
  }
```

Instead of doing this, Fable wrote three plans for three separate PRs. I wondered why something so simple would need three PRs. This second plan explained why. Instead of adding a jev call to the agency agent, Fable had written out a plan that involved modifying the built-in `llm` function, to add a new `firstToolCall` parameter:

```ts
const answer = llm(prompt, {
  tools: [researchAgent, codeAgent],
  firstToolCall: { name: "researchAgent", args: { userMsg: "..." } },
});
```

In Fable's own words:

> The call pushes the prompt onto the thread as usual. Then, where it would send the first request to the model, it acts as though the model had replied with exactly that one tool call. From that point nothing is different

There is so much wrong with this:

1. It is modifying a built-in function that is used everywhere
2. It's adding a new way to call functions. Instead of just calling the `researchAgent` function as agency code, it's getting called through this `firstToolCall` parameter. Introducing a new way to call functions is insane in any language, but especially in Agency, because Agency has a special feature that lets users pause anywhere in their execution and serialize the current execution state. Then they can resume execution, and Agency does a lot of work to get them back to the exact line where they left off execution. This feature is a ton of work, and this new way to call functions sidesteps all of it – basically leaving a core feature of Agency broken.
3. On top of it all, it's doing all this special casing for something that is completely unnecessary and for something that is essentially one use. As an analogy, in our kitchen, we have a knife, which is useful for all sorts of things. We don't have an apple corer, because even though it is great for coring apples, it is a tool that is only useful for that one task. We try to not have too many one-task tools – we try to have tools that are useful for many things. `firstToolCall` is a one task tool. It's making major modifications to core parts of agency, and that's not even that useful: It would only be useful in this specific use case, and even then, a match expression would be cleaner, easier to read, it would type check. This is worse on all counts.

To Fable's credit, I asked it:

> your second plan proposes firstToolCall. This is a horrible design. Tell me if you can figure out why.

It said:

```
⏺ Yes, I think I can. The option encodes a function call as a string and an untyped object, and passes it to llm() in order to not ask the
  model.

  llm(prompt, { tools: [...], firstToolCall: { name: "researchAgent", args: { userMsg: text } } })

  What the program means is this:

  researchAgent(text)

  Agency already has function calls, and in Agency a tool is a function. My design throws away what the language gives a call:

  - The type checker cannot see it. The tool name is a string and the arguments are an untyped object. A renamed subagent or a wrong argument
    fails at run time. The review had to add a test that every name in the route table is a real tool, and that test only exists because the
    design discarded a check the compiler does for free.
  - It fabricates a model reply. The thread gets an assistant message with a tool call that no model wrote. The plan then needed a new log
    field so a reader could tell the model did not choose the call.
  - llm() stops meaning "ask the model". With this option, a research turn makes zero model requests inside a function whose whole job is to
    make one. Memory recall, the call hooks and metering all had to be special-cased as "does not happen for this round".
  - It is a second way to control flow. Choosing what runs next is what if and match are for. I added a data-driven control mechanism beside
    the language's own, which is the mistake you have corrected me on before.

  I reached for it to avoid a real problem, and I put the fix in the wrong place. A handoff function called from code leaves its persona
  system message on the caller's thread, because the cleanup only happens when a model calls it as a tool. That is a gap in what a from-code
  handoff call does. I worked around it by making a from-code call pretend to be a model call.

  The design I should have proposed has the router pick a route, and the code then calls the subagent as a function:

  match (route) {
    "research" => researchAgent(text)
    "code" => codeAgent(text)
    _ => llm(prompt, mainOptions(turnStart))
  }

  The runtime work is then to make a handoff called from code clean up after itself on the caller's thread, which is where the problem actually is.
```

Correct. If I had not read these plans, however, Fable would have pushed a change that would have been worse in every way. It would have led to more complex code, more maintainability problems, and a new "feature" that silently breaks core features of Agency. _This_ is why I think agents cannot write code autonomously. I feel the response I always hear is, "Well, if it works correctly, who cares?" Which is fine when it's just building you a website. However, this is a programming language. It has tons of features, and when Fable proposes a new design like this, it does not go through every single feature to see how it would interact with this new design. And even if it did, the whole point of a programming language is that you can compose multiple features together, and it sure doesn't go through every combination of features to see how it would interact with this new design.

Fable 5.1 / Claude Code

## 10/27/26

In a session with Codex, I asked it to add support for tagging runs in agency. It discussed whether the tags should be simple strings or if they should be key-value pairs. It wrote up a plan. The plan involved setting key-value pair tags in the run's metadata. I asked if it is safe to put this in the run's metadata: If the users specify key-value pairs, is it possible that they would overwrite run metadata that we would want to keep?

Codex responded that this is safe to do, and that there is already precedence: Run metadata already has a `tags` field where we can store string labels, and a `custom` field, where we can store user-defined key-value pairs.
I then said, it looks like we have support for tagging already. We in fact have _two_ ways of tagging runs in Agency already: the plain string way and the key-value pairs way. So, what have we been designing this whole time?
Codex: "Yes. Both fields already exist, and I should have made that clear before proposing another one. Adding tagValues would create a third overlapping storage field. I now think that is unnecessary."

It is crazy to me that Codex saw that we already have two ways of tagging runs, but just took my request at face value and happily proceeded to write a plan for a third way to tag runs. Even after it showed me that there are already two ways of tagging runs, that didn't make it pause, reconsider, and say, maybe we shouldn't add a third way. This is an issue I have run into with coding agents several times before: they just do a really poor job of thinking through the assumptions being made in each conversation and challenging those assumptions.

GPT-6 Astra Ultra / Codex

## 10/26/26

In agency, we have a std::spill module (https://agency-lang.com/stdlib/spill.html). The idea is that if a tool call returns a result that is too long, we can use these tools to write the tool call result to a file and then give the agent some tools to read parts of the file selectively. This is a very common pattern. I recently found that the way Claude implemented this was that, for each tool call, it would check if the spill tools were part of the tools being given to the LLM, and change its behavior based on that. I think this is an anti-pattern. We should not be changing behavior based on what tools are being offered to an LLM. This was surprising behavior to me.

Opus 5.5/Claude

## 10/25/26

Claude wrote two plans. I asked it to execute the plans and create PRs for both. After it had done so, I Claude to merge the PRs into main and close the related tickets on Linear. Claude said we shouldn't close the tickets because the work hasn't been done. I asked it what the PRs were for then. It said that both PRs contained only the plan itself, and Claude had not actually executed the plan.

Opus 5.5/Claude

---

I've been asking Claude to track work in Linear for larger tasks. It creates milestones and tickets in each milestone and then works through the milestones. It was running into some weird dependency issues where it couldn't do a ticket in milestone 3 because it depended on work that had to be done in milestone 5. I looked at the milestones it had created. It turned out that it had not created the milestones based on how the work should be sequenced; it had essentially created milestones for grouping the tickets by topic. So for each milestone, it would completely build and polish a feature before moving on to the next one. This is sort of like painting a portrait by painting an extremely photorealistic eye, without spending any time planning the composition for the rest of the painting.

Opus 5.5/Claude

## 10/24/26

I worked with Claude to design a context layer package. I wanted users to be able to store docs in three different ways: in memory, as JSON files on disk, or in a SQLite database. I asked Claude to build something that supports all three. I glanced at the code and realized that it built config options for all three, but regardless of what option the user chose, the package always stored the data in SQLite.

I then told Claude that this wasn't correct and walked it through building a storage engine interface and three separate classes that implement that interface—one each for in-memory, files on disk, and SQLite. I asked it to implement this. I checked out its work and found out that, again, all three storage engines just used SQLite under the hood.

Fable 5.1/Claude
