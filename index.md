---
title: Fanfiction.net, Graphs, and PageRank: Oh My!
date: 2014-02-28
author: colah
mathjax: on
tags: math, fanfiction, graphs, visualization
---

Have you ever wondered about the connections between stories on fanfiction.net, a website where people write derivative stories of works they like? (Unless you're weird like me, no, you probably haven't.) Clearly, there are different groups with different opinions about what stories are good. There's a more complicated structure than just a scale of quality of stories.

Well, for some bizarre reason, I started analyzing fanfiction.net. To whet your appetite, here's a small visualization of Harry Potter stories on fanfiction.net as a graph:

<div class="bigcenterimgcontainer">
<img src="img/HP_union_size_larger.png" alt="" style="">
</div>
<div class="spaceafterimg"></div>


This blog post will explore the structure of the relationships between stories on fanfiction.net by constructing visualizations like the above, and much much larger ones. It will also provide story recommendations for many of the top users of fanfiction.net.



Introduction
-------------

Fanfiction is a wide-spread phenomenon where fans of different works write derivative stories. This ranges from young children writing their first stories about their favorite fictional characters, to professional-quality stories written by aspiring novelists. Many such stories are posted to websites where they are read by a large audience and commented on. The largest such website is [fanficiton.net].

The sheer amount of fanfiction out there is rather staggering. The total number of stories on fanfiction.net exceeds six million. Harry Potter stories account for around 14% of these, followed by Naruto (around 7%) and Twilight (around 4%) ([FFN Research](http://ffnresearch.blogspot.com/2010/07/fanfictionnet-story-totals.html)). The majority of these stories have very little in the way of readership, but popular stories can have a large number of readers.

Some research was done into the demographics of fanfiction.net users and other topics by [FFN Research]. They found that 78% of fanfiction.net authors who joined in 2010 identified as female. Further, around 80% of users who report their age are between 13 and 17.

A lot of other interesting research and analysis has been done on the blogs [Destination: Toast!] and [TOASTYSTATS].

In this post, we will examine the relationships between different Harry Potter stories on fanfiction.net. We will create visualizations, experiment with the application of Google's PageRank algorithm, and finally construct a crude recommendation tool. We will also discuss a number of directions for future exploration.

[fanficiton.net]:https://www.fanfiction.net/
[FFN Research]:http://ffnresearch.blogspot.com/
[Destination: Toast!]:http://destinationtoast.tumblr.com/stats
[TOASTYSTATS]:http://toastystats.tumblr.com/

Basic Methods
-------------

In addition to allowing users to post stories they write, fanfiction.net allows authors to "favorite" stories they like. Looking at which stories tend to be favorited by the same users gives us a way to understand connections between stories.

<div class="floatrightimgcontainer">
<img src="img/explanation.png" alt="" style="">
<div class="caption"></div>
</div>
<div class="spaceafterimg"></div>

In order to analyze this, we must collect a large amount of metadata from fanfiction.net ("scraping"). We note that we don't actually collect any significant content, just a lot of data about relationships between pieces of content. Fanfiction.net's terms of service, as the author understands them, allow this with some restrictions:

> 4(E) You agree not to use or launch any automated system, including without limitation, "robots," "spiders," or "offline readers," that accesses the Website in a manner that sends more request messages to the FanFiction.Net servers in a given period of time than a human can reasonably produce in the same period by using a conventional on-line web browser. Notwithstanding the foregoing, FanFiction.Net grants the operators of public search engines permission to use spiders to copy materials from the site for the sole purpose of and solely to the extent necessary for creating publicly available searchable indices of the materials, but not caches or archives of such materials...

In order to ensure compliance with these terms, the author intentionally built significant rate limiting into the scraper and took care to minimize the load put on fanfiction.net. While the issue of academic analysis was not mentioned, it was not excluded and fanfiction.net's operators have not previously objected to similar academic work. Further, this work could be the preliminary research needed for someone to build a good fanficiton search engine.

Another section of the terms of service prohibits collecting personally identifiable information, which they define to include usernames. As such, I have deliberately discarded all such information and don't use it. (Though, I note that several search engines do -- try searching for an authors name on any major search engine.) I do refer to some usernames in this post, but that was done entirely by hand.

In collecting data, since we are only looking at a subset of users, it is important to be wary of sampling bias. For example, if we sampled authors starting from the favorites of a particular author, or from those who had contributed stories to a community, we might get a very skewed perspective of the stories on fanfiction.net. The author considered a number of approaches, but concluded the fairest approach would be to use the authors of the most reviewed stories on fanfiction.net. This is a bias, but it should bias us towards the most interesting and important parts of the graph.

Graph Construction
------------------

A [graph], in the context of mathematics, is a collection of objects called vertices joined by connections called edges. For example, cities can be thought of as the vertices a graph connected by different highways and roads (the edges).

<div class="centerimgcontainer">
<img src="img/example-graph.svg" alt="" style="">
<div class="caption">An example of a graph (from Wikipedia)</div>
</div>
<div class="spaceafterimg"></div>

A weighted graph is a graph where some edges are "stronger" than others. For example, some cities are connected by giant 6-lane highways, while others are connected by gravel roads. Larger weights represent stronger connections and smaller weights represent weaker ones. A weight of zero is the same thing as having no connection at all.

We will be interpreting fanfiction as a weighted graph, where edges represent a "connection" between stories. We will be using as our weights for edges the probability that someone will like both stories, given that they like one. That is, $W_{a, b} = \frac{|F_a \cap F_b|}{|F_a \cup F_b|}$ where $F_s$ is the users who favorited the story $s$.

There are lots of other possibilities, some resulting in directed graphs:

* (directed) The probability that someone who favorites $a$ will favorite $b$:  $W_{a\to b} = \frac{|F_a \cap F_b|}{|F_a|}$
* The probability that someone who favorites $a$ favorites $b$ times the probability that someone who favorites $b$ favorites $a$: $W_{a,b} = \frac{|F_a \cap F_b|^2}{|F_a| * |F_b|}$
* The lesser of the probability that someone who favorites $a$ favorites $b$ and the probability that someone who favorites $b$ favorites $a$: $W_{a,b} = \min\left(\frac{|F_a \cap F_b|}{|F_a|}, \frac{|F_a \cap F_b|}{|F_b|} \right)$

Our experience was that it didn't matter too much for the results, for large graphs.

(It's worth noting that many of these could easily generalize to higher-dimensional edges for a weighted hyper-graph.)

In our selected weight definition, $W_{a, b} = \frac{|F_a \cap F_b|}{|F_a \cup F_b|}$, we give equal weight to the preferences of all users. But there's a lot of variance between users: some favorite everyting under the sun, while others very selectively favorite stories they really like. If we give the users who favorite thousands of stories the same weight as users who favorite ten, the users who favorite thousands dominate everything (and aren't a very good signal).

Instead, we give each user $u$ a weight of $\frac{1}{20+n(u)}$ where $n(u)$ denotes the number of stories $u$ has favorited. This results in a measure on the space of users, $\mu(S) = \sum_{u \in S} \frac{1}{20+n(u)}$,  and the equation for our weights becomes $W_{a, b} = \frac{\mu(F_a \cap F_b)}{\mu(F_a \cup F_b)}$.

Applying these techniques to a couple of the top Harry Potter stories, we get the following graph (using [graphviz]):

<div class="bigcenterimgcontainer">
<img src="img/HP-basic.png" alt="Small labeled graph of top Harry Potter stories" style="">
<div class="caption">Small labeled graph of top Harry Potter stories</div>
</div>
<div class="spaceafterimg"></div>

With a small amount of investigation, it's easy to understand a lot of the graph's structure. For example, on the lower right hand side, there's a triangular clique.

<div class="floatrightimgcontainer">
<img src="img/HP-basic-clique.png" alt="" style="">
</div>
<div class="spaceafterimg"></div>


A quick Google search reveals that this triangular clique consists of the "Dark Prince Trilogy" by Kurinoone. The stories are more strongly linked to their immediate predecessor/successor than the pair separated by a story are to eachother.

[graph]:http://en.wikipedia.org/wiki/Graph_(mathematics)
[graphviz]:http://www.graphviz.org/

Large Graph visualizations for Harry Potter
-------------------------------------------

If we use different tools, we can visualize much larger graphs.

We consider the top 2,000 most reviewed Harry Potter stories and their authors. Based on the author's favorite lists, we construct a weighted graph, with the stories as nodes (edge weights are calculated as above).

We then prune the graph's edges, keeping the top 8,000 most strongly weighted edges. We also prune the nodes, keeping only those with at least one edge. This leaves us with a graph of 1,623 nodes and 8,000 edges.

We then load this graph into the graph visualization tool [gephi]. We layout the graph using the OpenOrd and ForceAtlas2 layout algorithms. (OpenOrd was particularly good at extracting clusters. Beyond that, this was largely a matter of aesthetic taste.)

[gephi]:https://gephi.org/

<div class="bigcenterimgcontainer">
<img src="img/graph-HP-blank.png" alt="" style="">
<div class="caption">Graph of Harry Potter Fanfiction (top 1,623 stories)</div>
</div>
<div class="spaceafterimg"></div>


We can see lots of interesting structure in this graph: there are lots of clusters, some more connected than others.

A first hypothesis might be that some of these clusters are caused by language. As it turns out, this is the case:

<div class="bigcenterimgcontainer">
<img src="img/graph-HP-lang-labeled.png" alt="" style="">
<div class="caption">Graph of Harry Potter Fanfiction, colored by language</div>
</div>
<div class="spaceafterimg"></div>

Another cause of clusters may be the "ship" (romantic pairing of the story). Many readers have a strong loyalty to a particular ship -- for example, they might feel very strongly that Harry and Hermione should be together.

<div class="bigcenterimgcontainer">
<img src="img/graph-HP-ships-labeled.png" alt="" style="">
<div class="caption">Graph of Harry Potter Fanfiction, colored by ship</div>
</div>
<div class="spaceafterimg"></div>

(Note: Ships are inferred from tags story summaries. HP = Harry Potter, HG = Hermione Granger, GW = Ginny Weasley, DM = Draco Malfoy, SS = Severus Snape and LV = Lord Voldemort.)

One interesting point is that by far the most diffuse ship is HP/GW. It seems likely that this is because it is the ship we see in cannon Harry Potter, and so many stories not focused on romance default to it and unaligned readers are more tolerant of it.

One striking pattern in fanfiction is that a massive fraction of stories are male/male pairings. Such stories are frequently referred to as "slash." (For an exploration of why there is so much slash fanficiton, see this article.)

<div class="bigcenterimgcontainer">
<img src="img/graph-HP-slash-labeled.png" alt="" style="">
<div class="caption">Graph of Harry Potter Fanfiction, colored by slash</div>
</div>
<div class="spaceafterimg"></div>

Many stories include a slash tag in the summary. Some other stories tag themselves as "no-slash."

One interesting pattern is that stories tagged "no-slash" concentrate around parts of the border of slash stories. One possible reason may be that authors writing stories that might, from a glance at the summary or characters list, look like slash (for example, a story about Snape mentoring Harry, or Draco and Harry as friends) feel the need to explicitly signal that that is not the topic of their story.

The predisposition of the French cluster towards slash stories is interesting, but the cluster is so small I am hesitant to read anything into it.

You can also explore an [interactive graph of Harry Potter fanfiction](graphs/HP-ship/).

Large Graph Visualizations for Other Fandoms
---------------------------------------------

Of course, we can apply the exact same tricks to other fandoms.

**Naruto**

For example, Naruto is the second biggest fandom. Here's a graph of it:

<div class="bigcenterimgcontainer">
<img src="img/graph-NAR-blank.png" alt="" style="">
<div class="caption">Graph of top Naruto fanfiction (1,123 nodes and 4,000 edges)</div>
</div>
<div class="spaceafterimg"></div>

We can look at languages again:

<div class="bigcenterimgcontainer">
<img src="img/graph-NAR-lang-labeled.png" alt="" style="">
<div class="caption">Graph of top Naruto fanfiction, colored by language</div>
</div>
<div class="spaceafterimg"></div>

And also for ships:

<div class="bigcenterimgcontainer">
<img src="img/graph-NAR-ships-labeled.png" alt="" style="">
<div class="caption">Graph of top Naruto fanfiction, colored by ship</div>
</div>
<div class="spaceafterimg"></div>

**Twilight**

And again, we can graph the top twilight stories:

<div class="bigcenterimgcontainer">
<img src="img/graph-TWI-blank.png" alt="" style="">
<div class="caption">Graph of top Twilight fanfiction (1,031 nodes, 5,00 edges)</div>
</div>
<div class="spaceafterimg"></div>

We can color it by language:

<div class="bigcenterimgcontainer">
<img src="img/graph-TWI-lang-labeled.png" alt="" style="">
<div class="caption">Graph of top Twilight fanfiction, colored by language</div>
</div>
<div class="spaceafterimg"></div>

And by ship:

<div class="bigcenterimgcontainer">
<img src="img/graph-TWI-ships-labeled.png" alt="" style="">
<div class="caption">Graph of top Twilight fanfiction, colored by ship</div>
</div>
<div class="spaceafterimg"></div>

You can also explore an [interactive graph of Naruto fanfiction](graphs/NAR-ship/) and of [Twilight fanfiction](graphs/TWI-ship/).

PageRank
--------

What are the best fanfics on fanfiction.net? How can we identify them?

A naive approach would be to select the most favorited or reviewed stories. But people's quality of taste varies. A more sophisticated approach is Google's PageRank algorithm which is used to determine which web pages are of high quality.

In a normal vote gives equal weight to every voter. But some voters are better qualified to decide than others. In PageRank, we recalculate the votes again and again, giving each "person's" vote a weight based on how many votes they received in the previous step.

In the case of the Internet, we interpret a website linking to another website as  that website voting for the one it links to. We can similarly apply it to fanfiction by interpreting stories "voting" for the stories users are most likely to like, given that they like the voter.

**Harry Potter top stories by PageRank:**

<ol>
        <li><a href="http://fanfiction.net/s/1260679">Realizations</a> (16.4)</li>
        <li><a href="http://fanfiction.net/s/2636963">Harry Potter and the Nightmares of Futures Past</a> (15.7)</li>
        <li><a href="http://fanfiction.net/s/2318355">Make A Wish</a> (14.0)</li>
        <li><a href="http://fanfiction.net/s/5554780">Poison Pen</a> (11.7)</li>
        <li><a href="http://fanfiction.net/s/6413108">To Shape and Change</a> (11.5)</li>
        <li><a href="pagerank/hp.html"><b>More</b></a>
</ol>



**Naruto top stories by PageRank:**

<ol>
        <li><a href="http://fanfiction.net/s/2731239">Team 8</a> (11.1)</li>
        <li><a href="http://fanfiction.net/s/6694302">Naruto: Myoushuu no Fuuin</a> (6.42)</li>
        <li><a href="http://fanfiction.net/s/5409165">It's For a Good Cause, I Swear!</a> (5.57)</li>
        <li><a href="http://fanfiction.net/s/6051938">The Sealed Kunai</a> (5.24)</li>
        <li><a href="http://fanfiction.net/s/3929411">Chunin Exam Day</a> (5.14)</li>
        <li><a href="pagerank/naruto.html"><b>More</b></a>
</ol>


**Twilight top stories by PageRank:**

<ol>
        <li><a href="http://fanfiction.net/s/5100876">The Blessing and the Curse</a> (18.6)</li>
        <li><a href="http://fanfiction.net/s/4901517">Tropic of Virgo</a> (15.0)</li>
        <li><a href="http://fanfiction.net/s/5319052">A Rough Start</a> (12.7)</li>
        <li><a href="http://fanfiction.net/s/4769414">Creature of Habit</a> (12.6)</li>
        <li><a href="http://fanfiction.net/s/6550419">The Plan</a> (10.2)</li>
        <li><a href="pagerank/twi.html"><b>More</b></a>
</ol>

One neat thing we can do is give nodes on our graphs a size based on their PageRank. (We can also color nodes based on the first three components of the singular value decomposition of the adjacency matrix.)

<div class="bigcenterimgcontainer">
<img src="img/HP_union_size_larger.png" alt="" style="">
</div>
<div class="spaceafterimg"></div>



Story Recommendation
--------------------

There's something that's just begging to be done, at this point: story recommendations. Given our knowledge of what stories many users like, can we recommend other stories that they're probable to like?

This problem is called collaborative filtering, and is a well-established area. Unfortunately, it isn't something I'm terribly knowledgeable about, so I took a relatively naive approach: sum over the preferences of all users, weighted by how similar their preferences are to the user you are trying to predict.

Specifically, we give each story, $s$, a rank $R_u(s)$, for a user $u$. If the rank is high, we think $u$ is likely to like $s$.

$$R_u(s) = \sum_{v\in F_s \setminus \{u\}} \left(\frac{|S(u)\cap S(v)|}{20+|S(v)|}\right)^2$$

where $F_s$ is the set of users who favorited $s$ and $S(u)$ is the stories favorited by the user $u$.

For example, we can make recommendations for S'Tarken, the author of the most favorited Harry Potter story on fanfiction.net:

<ul>
	<li>*<a href="http://fanfiction.net/s/2559745">Learning to Breathe</a> (1.459)</li>
	<li>*<a href="http://fanfiction.net/s/2954601">Taking Control</a> (1.383)</li>
	<li>*<a href="http://fanfiction.net/s/1594791">Backwards Compatible</a> (1.381)</li>
	<li>*<a href="http://fanfiction.net/s/2636963">Harry Potter and the Nightmares of Futures Past</a> (1.377)</li>
	<li>*<a href="http://fanfiction.net/s/2479927">Harry Potter and Fate's Debt</a> (1.218)</li>
	<li>...</li>
</ul>

A * denotes that this is already one of the users favorite stories or one of their own stories. We can exclude their favorite stories, and their own stories:

<ul>
	<li><a href="http://fanfiction.net/s/2318355">Make A Wish</a> (0.949)</li>
	<li><a href="http://fanfiction.net/s/3401052">A Black Comedy</a> (0.750)</li>
	<li><a href="http://fanfiction.net/s/4536005">Oh God Not Again!</a> (0.679)</li>
	<li><a href="http://fanfiction.net/s/1260679">Realizations</a> (0.642)</li>
	<li><a href="http://fanfiction.net/s/2107570">Lord of Caer Azkaban</a> (0.635)</li>
	<li>...</li>
</ul>

These are all very popular stories. It's not very useful to S'TarKan if we recommend them extremely popular stories that they've almost certainly seen before. As such, it is interesting to penalize the popularity of stories.

Consider $\frac{R_u(s)}{|F_s|^k}$. When $k = 0$, it's our original rank. When $k = 1$, it full normalizes stories against popularity. And in between, it penalizes popularity to varying degrees. If we set k = 0.7, we get these recommendations:

<ul>
	<li><a href="http://fanfiction.net/s/2114122">Insanity</a> (0.034)</li>
	<li><a href="http://fanfiction.net/s/1995612">Shadow of the Serpent</a> (0.032)</li>
	<li><a href="http://fanfiction.net/s/2160456">The Bargain</a> (0.031)</li>
	<li><a href="http://fanfiction.net/s/1975479">Sinners</a> (0.029)</li>
	<li><a href="http://fanfiction.net/s/926568">Harry Potter and the Order of the Phoenix</a> (0.029)</li>
	<li>...</li>
</ul>

These are unpopular stories that similar users tend to like. Though, perhaps 0.7 is a bit too extreme.

Curious about what this algorithm would recomend for you? If you're a popular fanfiction author, you may be in my recommendations for top users for [Harry Potter], [Naurto] or [Twilight].

Since my scripts can't look at your author name while complying with fanfiction.net's terms of service, you will need to know your *author ID*. To get it, go to your fanfiction.net profile page and look at the URL. It will be of the form: `http://fanfiction.net/u/author_ID/...`. Then search for your author ID in the file!

[Harry Potter]:recs/hp.html
[Naurto]:recs/nar.html
[Twilight]:recs/twi.html

Conclusion
----------

In light of all this, I'd like to reflect on a few things.

**Big Data**: A year ago, I was very dismissive of "big data" as a buzzword. Primarily, it seems to be thrown around by business people who don't really understand much. But one thing I've learned in explorations of data like this one and working in machine learning, is that there is something very powerful about larger amounts of data. There's something very qualitatively different. The fanfiction data I used was actually quite small, only a few hundred users, because of how I limited the amount I downloaded, but I think it still demonstrates the sorts of things that become possible as you have larger amounts of data. (To be honest, a much more compelling example is the progress that's been made in computer vision using ImageNet... But this still influenced my views.)

**Digital Humanities**: Digital humanities also seems to be a bit of a buzzword. But I hope this provides a simple example of the power that can come from applying a little bit of math and computer science to humanities problems.

**Metdata and Privacy**: In this essay, we looked analyzed stories by looking at whether they were favorited by the same users. There's a natural "dual" to this: analyzing users by looking at whether they favorited the same stories. This would give us a graph of connections between users and allow us to find clusters of users. But what if you use other forms of metdata? For example, we now know that the US government has metdata on who phones who. It seems very likely that many companies and governments have information on where your cellphone is as a function of time. All this can construct a graph of society. I can't really fathom how much one must be able to learn about someone from that. (And how easy it would be to misinterpret.)

**Fanfiction Websites**: I think there's a lot of potential for fanfiction websites to better serve their users based on the techniques outlined here. I'd be really thrilled to see fanficiton.net or Archive Of Our Own adopt some of these ideas. Imagine being able to list a handful of stories in some category you're interested in and discover others? Or get good recommendations? The ideas are all pretty straightforward once you think of them. I'd be very happy to talk to the groups behind different fanfiction websites and provide some help or share example code.

**Deep Learning and NLP**: Recently, there's been some really cool results in applying Deep Learning to Natural Language Processing. One would need a lot more data than I collected, and it would take more effort, but I bet one could do some really interesting things here.

**Resources**: In principle, I'd really like to share my code and make it easy for people to replicate the work I described here. However, I think that would be really rude to fanfiction.net because it could result in lots of people scraping their website, and it seems likely many would remove my rate limiter. An alternative would be to share my extracted metadata, but, again, I think it would be really rude to do that without fanfiction.net's permission, and possibly a violation of their terms of service. So, in the end, I'm not sharing any resources. That said, all of this can be done pretty easily.





