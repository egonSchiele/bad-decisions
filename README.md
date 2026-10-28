# Bad decisions

I keep hearing from people about how I should not read an agent's code, and instead just focus on testing the interface. This hasn't worked for me personally. In this repo, I'm keeping track of concrete examples where agents have failed.

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
