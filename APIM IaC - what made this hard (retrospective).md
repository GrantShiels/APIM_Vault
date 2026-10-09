---
tags: [apim, iac, retrospective, delivery, lessons, devo-28368]
date: 2026-08-19
status: for-review
related-ticket: DEVO-28368
audience: "Tech Lead"
---

# APIM IaC - A big moan from Grant

> [!abstract] What this is
> This is a doc of me just airing out the major grievances I had during this process. I do want to note that it was an interesting  process and I got a lot of support and great communications from a lot of folks in a lot of different teams. I did enjoy learning this process but I still think it was poorly handled

| What made it hard | Root of it |
| --- | --- |
| The documentation I was given couldn't be trusted | Source material quality |
| There was no decision or planning record to inherit | Nothing was written down beforehand |
| This is a different discipline, not a different tool | Skills gap, unacknowledged in planning |

The single sentence version: I was handed an unfamiliar infrastructure product, unreliable
documentation, no prior decision or planning docs, and a very loose ask. 

---

## The guide I was given couldn't be trusted

The work was framed as "automate the existing setup guide". That guide is a manual runbook of **11
phases plus a separate alerting section**, and from my suspicion from talking to other people who worked closer with VP it might have been AI generated based on the previous APIM IaC component that I was told to ignore as it wasn't up to our quality or followed out patterns. So neither its structure nor its completeness could be assumed correct.

Before any judgement about quality, the sheer size of this guide was also pretty mental:

| Individual numbered steps                      | **114**  |
| ---------------------------------------------- | -------- |
| Once sub-steps are counted, individual actions | **~250** |
| Variables to define up front                   | **9**    |
| Prerequisites to have in place first           | **7**    |

The bigger problem was that nobody owned it. The guide contained a genuine number of errors, gaps and internal contradictions, there were steps that referred to things that didn't exist,
configuration that isn't valid as written, and instructions that contradicted other parts of the
same document.

Every one of those meant I needed to investigate the issue and find someone to ask. And most of the time there was no one to ask or no way to clarify the issue. No one internally owned the document and no one had been directly involved in writing it. 
I did got a lot of assistance from folks being helpful, rather than answers from anyone who knew actual solid info. I had to look through Slack threads or ask multiple people before I got answers, and even then a lot of the time it was guesswork or assumptions. There were several times where there were known issues in the guide but no one had updated it, folks had just left a comment in a Slack thread or on Confluence. **If you know there is an issue with something you fix it, or you let people know.** 

Every step became verify-then-build rather than build, and each unresolved contradiction became its own wee investigation. I had to read the guide, test the behaviour in Azure, work out what was
actually intended, then write down the answer nobody had. That is a fundamentally slower mode of working than "automate this document", which is what was costed.

---

## There was no decision or planning record to inherit

Apart from the guide there was no documentation around APIM. Nothing on why it was chosen, why the configuration is set up the way the guide describes, what decisions had been made or the reasoning behind any of them.

For any other piece of work at Dayshape there is an expectation of an FDP and planning documents that can be pointed to. They record what was decided, what was rejected, and why something was built one way rather than another. That is the standard, but for APIM there was jsut a big pile of nothing.

No single person knew the whole thing. There was nobody who could sit down and explain it end to end. Answering a straightforward question usually meant asking two or three different people and assembling a picture from the fragments, or going back through Slack threads from last year hoping the reasoning had been typed out at the time.

A lot of my time was lost just trying to piece together any sort of information from any scraps I
could find. I'd learn one thing from one person and then something else from another, then find a
random Slack thread from December last year or a random comment left in Confluence. It was very
frustrating.

Many of these decisions weren't mine to make either. They were questions that touched a bunch of
areas needing input from product, platform, SRE or others. A lot of them also weren't really
questions, they'd be like standard knowledge in other teams that I simply didn't know about, and in a few cases weren't documented anywhere. I did not have the knowledge or the skills to make the call, so each one had to be written up from nothing and explained to somebody who didn't know how APIM worked. It was a very convoluted process.

I was not involved in planning this piece of work and to be honest I'm still not fully sure who
was, from within Dayshape so this is an outside view. But compared with how our other features are set up, this is pretty poor. We are picking up the wider API work in Ecosystem, and if that has been planned and recorded the same way, we should expect to hit very similar issues to the APIM stuff.

---

## This is a different discipline, not a different tool

This is just my opinion by the way, happy to be talked down but just my 2 pennies. 

The work was sort of treated as allowing Ecosystem to upskill in Pulumi and IaC. I don't think it is
going to achieve that goal whatsoever. I think this was the completely wrong piece of work to try and upskill on, and now that I understand how an APIM is structured and how Pulumi fits together, I still think that. APIM is a large service that touches identity, networking, secrets, DNS and customer-facing content all at once. It is not a first introduction to IaC, it is a huge piece of work you would take on once you already know what you're doing, even then it's still pretty beefy. Something small and self-contained, a Logic App for example, would have been a lot better. 

I am not alone in that view. Multiple people across SRE, Infrastructure and Platform have said they
were surprised this was handed to a team with no IaC background, specifically because of how complex it is and how many areas it touches.

As it stands right now I honestly can't tell you if the approach in the PoC is correct. It works,
and it follows the standards in the codebase as far as I am aware, but the code has not been
reviewed by an infrastructure engineer. Its only review has been from a Copilot skill which has been
useful, but has also been very incorrect multiple times during this spike. That is not a review
standard that would be accepted anywhere else in the codebase, and it should not be the standard
here. How can other engineers or QAs in Ecosystem be expected to review this, or know anything about it? I'm happy to pass on some knowledge but even what I know I'm not sure if I trust based on everything else I've motioned in this doc. 

On a more personal note, I think it is crazy for the size of Dayshape that the same engineer is expected to write infrastructure code and fix CSS and HTML bugs. Those are very different knowledge domains and skillsets. It's so vast and I don't think this should be the standard. Maybe I am selling myself short or being overly anxious, but I think it could be a very dangerous as like I've said I'm not sure if what I've implemented is correct. 

From conversations I've had across teams, the impression was also that the responsibility of this piece work had been kicked about for a while without anybody wanting to touch it. Whether that's accurate or not I dunno but it's how it appears and it is worth knowing that is the impression I have been given from speaking to others.

---

## What would have made the difference

Below is a list of things that I think we should make note of to do next time before the work even starts

| What                                                       | Why it matters                                                                                                                                                                  |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **A named person for each specialist area**, not "ask SRE" | Parts of this needed expertise I don't have. "Ask a team" produces delay; a name produces an answer.                                                                            |
| **A real environment on day one**                          | The single highest-value unblock. Everything about confidence traces back to this. Also a doc or someone to show us how to deploy and what pipelines to use etc                 |
| **A short discovery step before estimating**               | An hour spent checking "is the documentation reliable? is there an environment? does a plan exist?" would have reframed the whole piece                                         |
| **Treat unfamiliar-discipline work as unfamiliar**         | Pair with someone who knows the discipline, and agree up front what review and testing should even look like                                                                    |
| **Decide the decisions first**                             | Sign-offs discovered mid-build are a scheduling problem. Found up front, they're an agenda                                                                                      |
| **Missing decisions or planning docs**                     | If we're going to be working with a big external team before we need to hold them to the same standard as we do internally. <br>We need decision docs, planning docs and an FDP |

> [!note] The one-line lesson
> The problems here weren't technical and they weren't capability. Almost every one of them was **lack of knowledge or written decisions/docs** and would have been cheap to establish. The biggest way to address this is to hold VP to the same standard as we do with internal teams. 

---

## What went well 

Set against the above, honestly:

- **Everything is written down.** 47 traps, the decision set, the deployment guides, the reasoning
  and the dead ends. That's the direct answer to the knowledge-concentration risk.
- **The hard parts are genuinely proven** against real Azure, not assumed.
- **The reversal on the existing code was the right call**, made on evidence rather than instinct.
- **Support from across the department has been great**, had a lot of people reach out to help and answer my questions. 

---

## Related

- [[Team skills vs APIM delivery - risk assessment]] — the forward-looking version of this: what
  could still go wrong, and what reduces it
- [[APIM IaC - known traps and gotchas]] — the 47 traps behind the documentation section
- [[APIM IaC - outstanding questions]] — the open questions behind the planning section
- [[_Decisions needing sign-off - start here]] — what still needs somebody outside the team
- [[Decision - do not build on existing APIM IaC]] — the reversal on the supplier code
- [[Existing APIM component - state assessment]] — the superseded assessment
- [[APIM IaC - rollout and testing strategy]] — where and how this gets tested
