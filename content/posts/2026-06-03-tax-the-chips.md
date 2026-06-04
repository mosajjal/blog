---
layout: post
title:  "Tax the chips, before the models"
comments: true
aliases: [/tax-the-chips]
date:   2026-06-03
tags:
- AI
- economics
- policy
images:
- /img/tax-the-chips/titlepic.png
description: Keynes promised our generation a 15-hour week. The surplus showed up right on schedule, it just went up instead of out. Tax the silicon, not the labs, and bank some of it back as time.
categories:
- AI
---

In 1930, with the world sliding into the Depression, John Maynard Keynes sat down and wrote something wildly optimistic. The essay was called *Economic Possibilities for our Grandchildren*, and the bet at the center of it was simple: machines were getting good enough, fast enough, that within a hundred years the economic problem would basically be solved. His grandchildren, he figured, would work about fifteen hours a week. The rest of the time they'd spend on life. Art, friends, leisure, the stuff you actually remember.

<!--more-->

![Tax the chips, before the models](/img/tax-the-chips/titlepic.png)

The hundred years is almost up. We're four years out from his deadline, and by the only measure that mattered to his argument, the wealth, he was right. We are staggeringly richer than 1930. The machines came, and then better machines came, and now machines that write and reason are arriving faster than anyone can keep up with. And we work forty-plus hours a week, same as our parents, more than some of them. Keynes got the production exactly right and the life completely wrong.

That gap is the whole story. The surplus he predicted is real. It just didn't go where he assumed it would. It went up.

## We did the math on this once already

Here's the part people forget when they treat the forty-hour week like a law of nature: it was extracted, not granted, and it cost real output to get.

Go back to 1870. A worker in the industrializing world put in something like three thousand hours a year. Sixty, seventy hours a week, most weeks of the year, in mills and mines and on factory floors. By the back end of the twentieth century that number had roughly halved. That decline didn't fall out of the sky because everyone got nicer. It got fought for, decade by decade, and the fight was ugly.

The rallying line of the eight-hour movement is almost a poem: eight hours for work, eight hours for rest, eight hours for what we will. In May 1886 something like half a million workers walked off the job across the country demanding exactly that. In Chicago it tipped into violence at Haymarket Square, a bomb went off, police fired into the crowd, and four organizers were later hanged on thin evidence. The backlash set the cause back years. It took the Depression, and then the Fair Labor Standards Act in 1940, to finally write the eight-hour day and the forty-hour week into American law. Ford had already jumped early, putting his factories on a five-day week in 1926 after working out that the sixth day bought him almost nothing in output.

So when someone says the forty-hour week is sacred, I think they have it backwards. It was a settlement. Society looked at a massive productivity shock, the steam-and-electricity one, and made a deliberate choice to bank part of that surplus as time instead of taking all of it as output. It cost something. We decided the lives were worth more than the extra production. And we were right.

That's the precedent that matters for AI. Not the specific number, the move. When the machine starts doing the work, you get to decide who keeps the hours it saves.

## So why are we still working five days?

The AI productivity story is real enough that I'd rather stop litigating whether it's happening and start arguing about who keeps the money. Work that ate hundreds of hours now takes ten. And the gain tracks almost linearly with two things: how many tokens you burn, and how smart the model is on the other end of them. Concrete. Measurable. Which makes it exactly the kind of surplus you can choose to redistribute, the way we redistributed the last one.

So the first question isn't "how do we tax this." It's the one Keynes already asked. If we shortened the week off the back of steam and the assembly line, why not off the back of this? A three or four day week. If AI is producing the gains everyone says it is, there's no reason the entire surplus should trickle into share buybacks and executive comp while the person whose job got compressed works the same Monday-to-Friday they always did. The week is a dial. We've turned it before.

But leisure has to be paid for, and this is where fairness gets genuinely hard. The productivity is coming off people. The layoffs happen first to fund the buildout, then again because the AI does the work that the laid-off person used to. If the surplus just consolidates at the top, you end up with precisely the world we're in: Keynes's wealth without Keynes's leisure. So how do you redirect a slice of it back down without breaking the thing that's creating the value in the first place?

## Don't tax the miners

There's a lot of noise right now about taxing "AI companies," and almost all of it points the gun at the wrong target.

Think about the California gold rush. The story everyone remembers is the forty-niners striking it rich, but most of them didn't. The man who got rich was Samuel Brannan, who never dug for an ounce. He bought up every pick, pan, and shovel in San Francisco before announcing the discovery, then sold pans he'd paid twenty cents for at fifteen dollars apiece. Shovels went from a dollar to thirty. He became California's first millionaire selling tools to dreamers. Levi Strauss showed up a few years later and got rich selling them the pants. The miners took the risk. The suppliers took the money.

The AI market is that, almost exactly. OpenAI and Anthropic are the miners. They're pouring billions into the ground and most of it is still a hole. OpenAI's own projections have it losing money deep into the back half of the decade and not breaking even until past 2030. Anthropic is running leaner and is aiming for profitability around 2028, but it isn't there today either. Tax these companies and you slow down the only thing actually producing the productivity gains, in a knife-fight of a market where the competition is the point. And nobody has the political appetite to clamp down on frontier AI anyway, because the whole thing is now wrapped in national security and great-power politics. That fight is done. The money was never there to begin with.

The money is in the shovels.

Nvidia runs gross margins north of seventy percent, and the per-unit markup is almost comic: an old 2023 teardown put the profit on a single H100 somewhere near eight-to-ten times what it costs to build. And Nvidia doesn't even make the things. It's fabless. It designs, and TSMC fabricates, and TSMC posted a net profit margin around forty-five percent in 2025, the best year in its history. The club that can actually etch leading-edge silicon is tiny: TSMC, Samsung, and the Huawei-and-SMIC stack on the Chinese side. A handful of companies on the entire planet can do leading-edge tokens-per-second at scale. That concentration isn't a footnote. It's exactly what makes the layer taxable.

## A levy on anything that makes a token

So here's the proposal. Put the tax on the hardware, per chip.

When a GPU is manufactured, estimate what it'll do over its life, call it three to four years for a datacenter part. Roughly how many trillions of tokens it'll generate, how much useful intelligence that represents, set against average utilization, average power draw, average contribution to the economy. Turn that into a single number: this is what this piece of silicon is expected to add to GDP. Then tax against that number, spread across the chip's lifetime.

I'm not going to pretend this is clean. It's a guessing game with a lot of moving parts. The models change what a chip is worth almost monthly. An M3 in someone's laptop runs maybe three hours a day; an H100 in a datacenter runs closer to twenty. Power costs swing wildly by region, different workloads feed the economy differently, and on and on. But it's a reasonable place to start, and the reason I like it is that it doesn't lean on the SaaS layer at all.

That's the crux. If you only tax AI-as-a-service, two bad things happen. People route around it by pushing inference onto local hardware, and you end up taxing the frontier labs whose products are your Gemini, your Opus, your GPT, the exact companies you don't want to kneecap. Tax the hardware instead and you stop caring. The cost lands in the capex of every datacenter before a single API call gets billed. An H100 might cost double what it does today. Fine. Nvidia is making multiples on those chips and demand isn't going anywhere for the next five to ten years. Whether Colossus, the cluster xAI threw up in Memphis (a hundred thousand H100s at launch, since scaled past two hundred thousand), runs five billion dollars or ten is almost irrelevant to the people building it if the upside is what they say it is.

Hardware has one more thing going for it: it's physical. You can trace a supply chain for a chip in a way you simply cannot for an inference call ricocheting across six jurisdictions. That makes it governable. Countries get to decide for themselves whether to tax low to attract the fabs and the clusters or tax high to fund the redistribution, and they get to weigh that against their own national interest, because the taxable thing is an object sitting on their soil. Try drawing those lines around cloud LLM calls crossing regions and you'll spend a decade writing provisions that leak the moment you finish them.

There's real cleanup left on the cross-border SaaS question and I don't want to wave it away. But I think most of it solves itself. If the capex of building an inference datacenter already carries the tax, it stops mattering much whether the SaaS in front of it is making or losing money. You already collected at the layer where the money actually lives.

## Back to Keynes

Keynes got the wealth right and the leisure wrong for one reason: he assumed the surplus would distribute itself. It doesn't. It never has. Last time, it took strikes and a bomb and a federal law and forty years of arguing to pry a shorter week out of the industrial boom. The leisure he promised was sitting right there in the numbers the whole time. We just had to go take it.

This time the surplus is sitting in the silicon, in a few dozen fabs and a handful of companies printing the best margins on earth. We can see exactly where it is. So tax the chips, before the models. Shorten the week off the back of it. Give people back some of the time the machines were always supposed to buy them. That was the deal Keynes thought we'd struck a century ago. We can still make good on it.
