# Know Enough to Be Dangerous

I didn't write most of the code in kindling. An AI did. And I think that's the most interesting thing about it.

Not because "AI wrote my code" is novel — at this point, everyone's doing it. What's interesting is *why it worked*. Because frankly, for most of my career, it wouldn't have.

---

## The prep you don't notice

I spent years adjacent to infrastructure. I was at GitHub when Actions launched. I worked on developer tools. I sat in meetings where people argued about CI runner architectures and webhook delivery guarantees and container registry auth flows. I absorbed the shape of these problems without ever being the person who solved them with code.

When I sat down to build kindling, I didn't know how to write a Kubernetes operator. I'd never used controller-runtime. I couldn't have told you how to wire up a Custom Resource Definition from scratch. But I knew what one was. I knew *why* you'd want one. I knew the difference between a reconciliation loop and a watcher, conceptually, even if I'd never implemented either.

That turned out to be everything.

---

## The actual workflow

Here's what building with AI looks like when you have domain knowledge:

**I say:** "The operator needs to auto-inject DATABASE_URL when a service declares a postgres dependency."

I don't say: "Write me a Kubernetes operator." I don't say: "What's a good architecture for this?" I know the exact behavior I want because I've seen it done — in Heroku, in Railway, in a dozen PaaS tools I've used or studied. I'm describing a feature, not asking for a tutorial.

**I say:** "Secrets set after deploy need to restart the pods that consume them."

I know this because I've been the developer whose environment didn't pick up the new secret. I've been the person filing the support ticket. The requirement doesn't come from a spec — it comes from scar tissue.

**I say:** "The tunnel needs to save the original TLS config as an annotation before stripping it, so we can restore on cleanup."

I know Kubernetes annotations are the right place for this because I've seen the pattern. I couldn't write the JSON patch from memory, but I know the *strategy* is correct. The AI handles the implementation. I handle the "what" and "why."

---

## What I couldn't have done

Let me be honest about the gap. Kindling is about 15,000 lines of Go across an operator and a CLI. It has 74 controller specs, e2e tests that spin up real Kind clusters, a React dashboard, Helm chart generation, multi-arch container builds, and a production deploy pipeline.

If I sat down with the Go docs and started typing, I'd finish maybe never. Not because I can't learn Go — I can, I have — but because the sheer surface area of this project would bury a solo developer. Operator SDK semantics. Kustomize overlays. Kaniko build contexts. Cloudflared tunnel orchestration. Crane image copies. The list doesn't end.

The AI collapses the implementation time. But it doesn't collapse the *decision* time. Every architectural choice — using annotations for tunnel state, building amd64 in the CI runner instead of cross-compiling on the Mac, prioritizing the in-cluster registry over the Docker daemon for snapshots — those came from me knowing what would break if we did it wrong.

---

## The dangerous part

"Know enough to be dangerous" is usually self-deprecating. You say it to hedge — *I'm not an expert, but...*

I'm reclaiming it. Dangerous is exactly right. If you know enough about a domain to describe the behaviors you want, to recognize when an implementation is wrong, and to steer toward patterns that actually work in production — that's dangerous. You can ship real software. Fast. Alone.

The corollary is also true: AI without domain knowledge produces plausible garbage. It'll write you a Kubernetes operator that looks right, passes a vibe check, and falls apart the first time a pod gets evicted during a rolling update. If you can't smell that bug coming, you'll ship it.

---

## What this means for builders

The leverage isn't "AI writes code." The leverage is: **domain knowledge + AI = products that would otherwise require a team.**

I'm one person. Kindling has the feature surface of something a small platform team would build. That's not because I'm faster than a team — it's because the bottleneck was never typing speed. It was always knowing what to type.

If you've spent years in a domain — infrastructure, fintech, healthcare, whatever — you're sitting on the most valuable input an AI can receive: *informed intent*. You know the problems. You know the edge cases. You know why the obvious solution doesn't work.

That's not "enough to be dangerous." That's the whole game.
