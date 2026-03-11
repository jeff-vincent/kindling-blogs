# Know Enough to Be Dangerous

I didn't write most of the code in kindling. An AI did.

That's not a flex. At this point everyone's using AI to write code. What I keep thinking about is why it actually worked — like, produced a real thing that runs in production and does stuff that's genuinely hard. Because the same tools are available to everyone, and most of what comes out of them is mid.

---

## Where the phrase comes from

My first lead engineer told me I "knew enough to be dangerous." He meant it as both a compliment and a warning. I could build things — I had the instincts, the product sense, the ability to look at a problem and see a path through it. But what I built was ad hoc. I'd reach for whatever worked instead of the pattern that would hold up over time. He told me it was just a matter of time. Keep building, keep learning, and eventually the knowledge would stack up enough that the "dangerous" part would just become "good."

He was right, mostly. It just took a weird path to get there.

I spent a few years at MacStadium building open-source developer tools — a Python SDK, GitHub Actions integrations for ephemeral CI runners, that kind of thing. I live-demoed for teams at PyTorch and Disney, which sounds impressive but mostly meant I was the guy nervously refreshing a terminal on a webinar. From there I went to Velocity, a Kubernetes startup, where I built end-to-end demo apps — microservices, MLOps pipelines, event-driven architectures, GPU-backed workloads — to show what the platform could do. Then MongoDB, where I embedded with the engineering team building their Kubernetes operator (MCK). I deployed it, broke it, documented it, built internal tooling around it, and generally spent my days living inside the operator lifecycle.

None of those were "senior engineer" roles. I was a technical writer. DevRel. The person who builds the demo and writes the docs. But I was always building — deploying clusters, wiring up CI, debugging why a pod wasn't scheduling. I built a whole Kubernetes-native dev platform on my own (Labbrly) just to see if I could: JWT auth mapped to namespaces, async Python microservices, Redis queues, a compute API for interactive environments. It worked. It was also, in my first lead's words, ad hoc as hell.

The point is: by the time I started kindling, I'd spent years accumulating context about how Kubernetes, CI/CD, and developer platforms actually work. Not from reading docs — from deploying things, watching them break, and figuring out why.

When I started building kindling, I'd never written a Kubernetes operator in Go. Never used controller-runtime. Couldn't have wired up a CRD from scratch. But I knew what a CRD was and why you'd want one. I knew the difference between a reconciliation loop and a watcher, even if I'd never implemented either.

The gap between "I know how I'd do it" and "I can actually do it" had always been there. It just got a lot smaller.

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

## The gap is closing

For most of my career, "I know how I'd do it" was where things stopped. You have the vision, you understand the problem, you can describe the solution in detail — and then you either need a team to build it or six months you don't have to skill up on the implementation details yourself.

That gap is disappearing. AI collapses the distance between knowing what to build and actually building it. And tools like kindling exist on both sides of that equation — it's a product born from this way of working, and it's a product that makes this way of working easier for everyone else.

Kindling does stuff that normally takes a platform team. It runs your whole dev environment on a local Kind cluster. Builds with Kaniko, deploys with a single command, handles secrets, tunnels, production graduation with Helm charts. It's a real product that solves a real problem. And I built it alone because I'd spent enough years in this space to know exactly what it should do.

If you've been in a domain long enough to have opinions about how things should work — not just preferences, but informed opinions from watching things break — pair that with your favorite AI agent and go build something. The "I know how I'd do it" era is over. Now you can just do it.

That's what "know enough to be dangerous" actually means now. Not a gap between knowledge and ability. Just the starting line.
