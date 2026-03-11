# Know Enough to Be Dangerous

I didn't write most of the code in kindling. An AI did.

That's not a flex. At this point everyone's using AI to write code. What I keep thinking about is why it actually worked — like, produced a real thing that runs in production and does stuff that's genuinely hard. Because the same tools are available to everyone, and most of what comes out of them is mid.

---

## The background that mattered

I spent years around infrastructure without being the person writing it. I was at GitHub when Actions launched. I worked on developer tools. I sat in rooms where people debated CI runner architectures, webhook delivery semantics, container registry auth. I understood the shape of those problems. I just wasn't the one solving them in code.

When I started building kindling, I'd never written a Kubernetes operator. Never used controller-runtime. Couldn't have wired up a CRD from scratch. But I knew what a CRD was and why you'd want one. I knew the difference between a reconciliation loop and a watcher, even if I'd never implemented either.

That turned out to matter more than I expected.

---

## What it actually looks like

When I'm working with AI, I'm not asking it to figure out the architecture. I'm telling it what I need, specifically, because I've seen the problem before.

"The operator needs to auto-inject DATABASE_URL when a service declares a postgres dependency." I know the behavior I want because I've used Heroku. I've used Railway. I've seen how this should work from the user's side. I'm not asking for a tutorial — I'm describing a feature.

"Secrets set after deploy need to restart the pods that consume them." I know this because I've been the developer whose environment didn't pick up the new secret. That requirement comes from scar tissue, not a spec.

"The tunnel needs to save the original TLS config as an annotation before stripping it, so we can restore on cleanup." I know annotations are the right place for this because I've seen the pattern. I can't write the JSON patch from memory. But I know the strategy is right, and the AI handles the rest.

---

## The honest part

Kindling is roughly 15,000 lines of Go across an operator and a CLI. It has 74 controller specs, e2e tests that spin up real Kind clusters, a React dashboard, Helm chart generation, multi-arch container builds, and a production deploy pipeline with TLS.

I could not have built this alone. Not in a reasonable timeframe. Not because I can't learn Go — I can — but because the surface area is enormous. Operator SDK. Kustomize overlays. Kaniko build contexts. Cloudflared tunnels. Crane image copies. It just keeps going.

The AI collapses the implementation time. But it doesn't collapse the decision time. Every architectural choice — annotations for tunnel state, building amd64 in the CI runner instead of cross-compiling on the Mac, prioritizing the in-cluster registry over the Docker daemon for snapshots — those came from knowing what would break if we got it wrong. The AI doesn't know that. You have to.

---

## The flip side

AI without domain knowledge produces stuff that looks right and isn't. It'll write you a Kubernetes operator that passes a vibe check and falls apart the first time a pod gets evicted during a rolling update. If you can't smell that coming, you'll ship it.

That's the thing nobody talks about with AI-assisted development. The tool is only as good as the person driving it. Not in a "prompt engineering" sense — in a "do you actually understand the domain you're building in" sense.

---

## What I'm getting at

Kindling does stuff that normally takes a platform team. It runs your whole dev environment on a local Kind cluster. It builds with Kaniko, deploys with a single command, handles secrets, tunnels, production graduation with Helm charts. It's a real product that solves a real problem.

And I built it alone, with AI, because I'd spent enough years in this space to know exactly what it should do. Not how to implement every piece — but what the right behavior was, what the failure modes looked like, what the user actually needed.

If you've been in a domain long enough to have opinions about how things should work — not just preferences, but informed opinions born from watching things break — you're sitting on something valuable. The AI can write the code. But it needs someone who knows what the code should *do*.

That's what "know enough to be dangerous" actually means. Not a hedge. Just the truth about how this stuff gets built now.
