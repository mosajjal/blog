---
layout: post
title:  To Catch a Rising (AI) Star
comments: true
aliases: [/catch-a-rising-ai-star]
date:   2025-04-04 00:00:00
tags:
- misc
images:
- /img/ai/catch-a-rising-ai-star.jpg
description: Why Your Current Strategy is Already Obsolete and How to Catch the Rising Stars Before They Leave You Behind
categories:
- misc
---

{{< rawhtml >}}
    <br>
    <audio controls>
    <source src="/img/ai/catch-a-rising-star.mp3" type="audio/mpeg">
    Your browser does not support the audio element.
    </audio>
    <br>
{{< /rawhtml >}}

If your day-to-day life is primarily behind your monitor and keyboard, this article is for you. AI is coming fast for our jobs, and most of us are not as alarmed as we should be, nor know what to do about it. I've spent the last 12 months digging deeper and deeper into Generative AI, spent 100s of hours building tools with all the "standard" architectures, and I've come to some conclusions that I'd like to share with the rest of us. I'm hoping this gives you the point of view that I think we should all have.
How Good is Generative AI, really?

When I got invited into Github Copilot's private preview in 2021, I immediately jumped in and got to testing. It was built on top of Codex, which was a "modified" version of GPT3. It had a lot of limitations, it didn't understand your code properly, and it was more of a distraction than a useful feature. It also didn't understand your codebase, so you were incentivized to build your code in giant one-filers. When I was working on anything serious at that time, I would turn off my copilot so I could focus and not get distracted by the constant wrong and/or out-of-context suggestions.

I held my pessimistic point of view around Generative AI through the advancements of GPT, and up until mid-2024. To me, it looked like the chatbot was pretending to know something, but didn't really know it. When pressed, the chatbot would just make something up, or refer to non-existent articles. There were also lots of papers around the fact that this problem in AI is unsolvable, which all turned out to be eventually wrong and "grounding" has become better and better over time.

Up until last year, AI agents were suffering from multiple deficiencies:

- Good models were extremely expensive

- Great models had short context windows

- Cheap commercial models were terrible

- Smaller (self-hosted) models were laughably bad at the basics

In just under a year, almost every single one of these issues is not only eliminated, but has improved beyond what I thought possible. For example:

- Distillation-based model training hits the market: A 300B parameter model is hard to self-host, but a distilled 32B parameter can be run on consumer hardware with 90-95% accuracy of the big model. I will refer to this as "Smaller Models are becoming Good."

- Reinforcement learning: This training method resulted in one of the most amazing breakthroughs of AI: Deepseek. Deepseek showed the world that with a fraction of the cost, you can train a state-of-the-art (SOTA) model. I will refer to this as "Good model weights becoming free."

- Reasoning: Introspection unlocked a new avenue to increase "intelligence." What I found with reasoning models is the fact that they tend to overthink, and they have a different "personality." This is particularly useful in Agentic use of AI. I will refer to this as "Models are more trustworthy."

- Tool Use / Function Calling (sometimes referred to as Model Context Protocol or MCP): Getting AI to use other tools is getting way easier, and this capability is huge. Props to Anthropic for pushing powerful APIs for this, and it's awesome how the whole AI world jumped on board, building similar capabilities. This gives AI arms and legs to actually do things. I will refer to this as "Models can now act on your behalf."

- Context caching and long context windows: The models went from 64k token context window, to 2M. This means that you can ask a question from an AI agent, and have it read a few books while it's answering your question. This alone has helped the coding agents to become very reliable. I will refer to this as "Models know your requirements." 

As you can see, within 12 months, AI went from a fun party trick to a serious product. AI companies famously don't make good money. But I think that's about to change.
What does good AI mean for me and my company?

I speculated around 5 areas that AI is trending towards. I can only assume the good trends above are only getting better. I don't see a future where the 2M token context window gets reduced to 1M; rather, I can easily see that token window increasing over time.

What I can see a lot of companies are [mistakenly] doing is to try and meet AI where it is now. Worse yet, some companies are starting their AI journey with what AI was capable of 2 years ago since that's an "established" architecture. Which brings me back to the title of this article: "To catch a rising star."

How do we build something that is going live in 6 months, and make sure it is not instantly a legacy product?
Do not rely on model's expertise and personality

Let's talk through this with an example. Let's say you're building a math chatbot. You first go to the AI leaderboard in math, choose the top one, and build your chatbot and it works decently well. I'm here to tell you, that's not a future-proof design. What you need to do is to build your chatbot with the #3 or #4 ranked in that benchmark, and contextualize it in a way that it can (almost) reach the level of #1 for your use case. Then, you can swap that model with #1.

Remember, benchmarks are always skewed and unreliable. If the model has seen the test material before, it can cheat itself through and gain a #1 spot. Doesn't mean it's good at what you want it to be good at. So your architecture should cater for that, and eliminate over-reliance on the model.
RAG, Vector Databases and Embedding models are a workaround, not a solution

This is going to be a hot take. In essence, Vector Databases are non-intelligent. In the RAG model, they act as a middleman to add the correct "context" to your prompt, to make sure the results are grounded. This has a fundamental issue that I refer to as the "bad assistant" issue.

To understand RAG and MCP, I always make the example of going to the doctor to get diagnosed. Let's imagine you're going to a diagnostician with a headache. The doctor (SOTA model) is very capable and knows what he's doing, but he cannot remember every single patient's medical history. So how do we set up this doctor's office to make sure the medical history is available? There are two main options:

Hire an assistant (RAG): 

We put a middleman before going to the doctors. First, you go to the assistant, give it your name and information, and it will write down all your relevant medical history on a piece of paper. Hands it to you, and tells you to show it to the doctor when you arrive. Remember, this is not your full medical history, but only the stuff that the assistant finds "relevant" to your headache. It might even be another patient's data since this assistant is really disorganized.

Teach the doctor how to use a computer (MCP/CAG): 

In this approach, you will visit the doctor first, and the doctor asks for your information. Then, the doctor will search your name inside the patient record database and pull the info they need to make an informed decision. After your diagnosis, the doctor then writes back your latest visit to the same database and sends you home.

Not only is MCP a clear winner in this scenario, but it also has another advantage: scalability. Using the above scenario, you have practically infinite doctors, but you still only have one assistant. The intelligence, context and scalability bottleneck will sit with the assistant. And if you didn't catch my hint in the example, MCP allows your data to remain evergreen since you're most likely going to the original source rather than a [possibly out of date] Vector database. RAG needs constant maintenance to keep the data up to date and re-indexed.

From the RAG architecture, I intentionally left out one segment: Data Chunking. While some RAG parts can be replaced by MCP, some parts of it are still essential. Data Chunking is one of them. Data Chunking allows your CAG costs to remain low, while giving you the ability to invalidate the caches of irrelevant data cheaply and quickly. There's an awesome article in AWS talking through RAG vs CAG and why CAG is gaining momentum:

https://community.aws/content/2v0HnXk5EuYF28u8G6WP9PI7kRL/rag-vs-cag-navigating-the-evolving-landscape-of-llm-knowledge-augmentation-on-aws

In particular, I would like to draw your attention to the 3 drawbacks of CAG, and why my previous comments around the advancements of AI are crucial:

Context Window Limits: 

CAG is limited by the LLM's context window size. As mentioned above, context window is always growing, and it's always a safe bet to assume 2-4M context windows are readily available. For extremely large document libraries, correct chunking will help solve this issue.

Cache Invalidation Cost: 

AIs are getting cheaper by the day, so this cost is bound to go down. Chunking allows you to invalidate parts of your context, not all, and help reduce this overhead.

"Cold Start" Latency: 

This time has been reduced over time. Moreover, RAG complexity and concurrent user scalability also creates significant latencies. Chunking keeps your context shorter which helps with this as well.
Reasoning models are over-thinkers

In 2025, Models are used in two main categories:

Suggestion: 

Chatbots, code completion, text summary, CAG/RAG.

Acting: 

AI Agents, MCP Clients, Code Execution, REPL.

Depending on which type of work you expect the AI to perform, one model could be better than the other. In my experience, in April 2025, and not knowing what the future looks like, I like to use reasoning models for actions, and non-reasoning models for suggestions.

People have suggested that Claude Sonnet 3.5 does better at coding than Sonnet 3.7 Reasoning. I have tested this personally, and I can see Sonnet 3.7 overthinking the issue to the point that the proposed suggestion is not an optimal one anymore. Conversely, I have seen AI Agents that are built to act or live-test code underperform using a non-reasoning model compared to a reasoning one.

The question I ask myself in order to choose a reasoning or non-reasoning model is simple: How much do I want my query deliberated before the answer? We all have that one friend... the one that looks at a problem and comes out with a really bizarre solution for it. When you ask them how did they arrive at that conclusion, they will lay it out in front of you and you can see how overthinking can just ruin good 'intuition'.

Now, I will admit that the above statement needs to be taken with a grain of salt. The evidence of practical difference between the two is anecdotal, and there are good signs that the effectiveness of these models might converge in the future.
