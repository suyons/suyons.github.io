---
title: "Small-Dataset Fine-Tuning - It Learned My Tone Long Before It Learned My Facts"
date: 2026-07-14
draft: false
tags: ["llm", "fine-tuning", "qlora", "unsloth", "evaluation"]
categories: ["Machine Learning"]
description: "QLoRA fine-tuning a 4B model on 90 Q&A examples fixed the base model's biggest problem — rambling English reasoning that never reached an answer — in exactly two epochs. It also taught the model to state wrong numbers with the same confidence as right ones."
showToc: true
---

## The setup

I QLoRA fine-tuned a Qwen3-4B base model on a small, doc-grounded Q&A set: 90 training examples, 10 validation, two epochs, all Korean-language questions about a public housing-loan subsidy program with a lot of specific numeric thresholds — income caps, deposit limits, interest-rate add-ons, deadlines. Before touching the training loop, I ran the same 10 held-out questions through the base model and recorded the answers, specifically so I'd have something to diff against afterward instead of just eyeballing "does this feel better."

The diff was more informative than I expected, in both directions.

## What changed immediately: it stopped thinking out loud without landing anywhere

The base model's failure mode wasn't wrong answers. It was no answer. Asked (in Korean) how the loan-extension period for a family with a newborn had changed, the base model spent its entire response budget reasoning in English and never reached a conclusion:

> Okay, the user is asking about the changes to the loan period extension criteria for families with children (including adoptive families). I need to provide a detailed answer. Let me start by recalling the previous regulations. First, I remember that in the past, the loan period for families with children was 10 years. But there was a change. The Ministry [...] The new criteria allow for a 15-year loan period. Wait, but I need to check the exact details. The key points are the eligibility criteria. For example, if the family has a child under 10 years old, they might qualify for the extended period. Also, the family's income level might be a

That's a real base-model quirk worth noting on its own: asked a Korean question, it defaulted to reasoning in English, and burned the whole token budget second-guessing itself instead of committing to an answer.

After fine-tuning, the same question got:

> The 'loan period extension criteria for families with children (including adoptive families)', effective from [date], has changed as follows: [...]

Direct, in Korean, no visible hedging. Across all 10 held-out questions, this pattern held: two epochs on 90 examples was enough to reliably suppress the rambling-without-concluding behavior and match the target answer style — language, directness, format. That's a bigger win than it sounds like on paper. An answer that's wrong but stated is something a reader can fact-check and correct; a 150-token block of reasoning that trails off mid-thought isn't usable at all.

## What didn't change: it got confidently, specifically wrong

Two of the ten held-out questions asked for numbers that were in the training set — not paraphrased, not requiring inference, just facts the model had seen during training.

**Which banks handle this loan?** Training data: *"Kookmin Bank, Hana Bank, and Shinhan Bank."* Fine-tuned model, asked the same question in a fresh session: *"The partner bank is **Kookmin Bank**."* One of three, stated as if it were the complete answer.

**What's the maximum loan amount?** Training data: *"Up to KRW 300 million (within 90% of the deposit amount), subject to the guarantee-issuance limit from the housing finance guarantor based on income."* Fine-tuned model: *"The maximum loan limit [...] is **KRW 150 million**."* Not a rounding error, not a partial truth — a different number the training data never states, delivered with the same confident, unhedged phrasing as every correct answer around it.

Both wrong answers read exactly like the right ones. Same tone, same directness, same lack of hedging. Nothing about the output signals "guessing" versus "recalling" — which is the actual problem, not the wrong numbers by themselves.

## The read

My working theory: with only 90 examples over two epochs, the model has many more chances to absorb the *shared* pattern across examples — answer directly, answer in Korean, don't narrate your reasoning — than it does to absorb any *individual* fact, most of which appear exactly once in the training set. Style is reinforced 90 times per epoch. A specific number is reinforced once, maybe twice, if it happens to recur in a related question. It's not surprising that the pattern reinforced 90 times wins early.

The practical consequence is worse than a model that hedges when unsure. A model that answers "I'm not certain, but I believe it's around 300 million" is safe to spot-check. A model that answers "The maximum is 150 million" with no hedge, in the same voice as every correct answer, is not — the fine-tuning made the wrong answers *harder to catch by tone*, not easier.

I want to flag how thin this evidence base is before anyone takes the theory further than I'm taking it here: 10 held-out prompts, no repeated runs to check variance, no automated factual-accuracy scoring — I read the outputs and compared them to the training data by hand. It's a real, reproducible observation on this run, not a benchmark result. Anyone trying to generalize "small SFT learns style before facts" as a rule should test it with a scored eval set and more than one seed before trusting it beyond this specific case.

## Takeaways

- At this scale — tens to ~100 examples, a couple of epochs — expect answer style (language, tone, directness, format) to converge well before factual grounding does. Don't assume a fine-tuned model that "sounds right" is answering correctly on facts that appear only a handful of times in the training set.
- If the numbers matter — pricing, deadlines, eligibility thresholds, anything with a wrong-answer cost — QLoRA-fine-tuning-in-the-parameters isn't the tool for making the model remember them reliably at this dataset size. Retrieval-augmented generation, where the exact figure is looked up rather than recalled from weights, is the safer place to put that responsibility. What this fine-tuning run actually bought was tone and format, not a lookup table.
