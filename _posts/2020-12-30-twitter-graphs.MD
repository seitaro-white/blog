---
title: "Exploring Social Graphs in Twitter"
date: 2010-12-30
tags: [graph theory, twitter, social networks]
header:
  image: "/images/perceptron/percept.jpg"
excerpt: "What does this bit do again?"
mathjax: "true"
hidden: true
---

# Article Title

### Introduction

[NEED 3 PARAGRAPHS HERE REALLY]
A few months ago I've became slightly obsessed with social network analysis/mapping, partially because as a recovering historian [...], but mostly because I just thought network graph visualisations look extremely cool. Inspired by this excellent bit of [graph analysis](https://towardsdatascience.com/generating-twitter-ego-networks-detecting-ego-communities-93897883d255), I decided to look into my tw


### Why am I doing this

### Background - what are social networks
### Network visualisation, some examples 

### Getting Data out of Twitter
Accessing data on twitter is fairly straightforward - of the big social media services, Twitter is the one that makes it easiest to get user data out programatically (for free anyway). All you need to do is [apply for a developer account](https://developer.twitter.com/en/apply-for-access), give a brief explanation of what you intend to use the API for and promise not to do anything naughty as per the T&C's - I had my account approved within a few hours. 

The API allows you to do all sorts of things like automatic tweeting and content creation (you might have seen various automated twitter bots on your feed before, such as this [Rejected Petitions bot](https://twitter.com/rejectpetitions) that tweets every UK parliament petition rejection (a personal favourite is this one to [put chicken livers back on the Nandos menu](https://petition.parliament.uk/petitions/563000)) but for social network analysis I'm just interested in pulling information about users out of twitter. The Twitter API allows you to pull data on any (non-protected) user such as profile information, basic stats like total number of tweets or likes. Crucially for social network analysis, we can get list of user 'followers' (accounts following that user) and user 'friends' (accounts being followed by that user). We can make things even easier by using [Tweepy](http://docs.tweepy.org/en/latest/index.html), a python wrapper around the API that handles things like pagination and errors - getting a list of friend IDs for a user can be accomplished with a simple function like this:

```python
import tweepy
def get_friend_ids(user_id):

    friend_ids = []
    try:
        for page in tweepy.Cursor(api.friends_ids, id=user_id, count=5000).pages():
            friend_ids.extend(page)

    except tweepy.TweepError as e:
        print(e)
    
    return friend_ids
```


Twitter does impose rate-limits on how many requests we can make to the API. Free users can request the data for up to 15 separate users every 15 minutes, which means we can't obviously do anything really large-scale (getting the data to generate a complete follower network of a famous account like [Taylor Swift](https://twitter.com/taylorswift13) with 87.8 million followers for example would take around 167 years). One thing we can do however is to extract a "ego-network" limited to a single level of separation from a single user - the nodes of a ego-network are limited to all the accounts that this single user followers, while the edges are all the follower relationships between those accounts. What this gives us is a tiny view of Twitter's vast social network from the perspective of a single person, a little community of users that they directly know about and follow and which we can study.

Since the average account (excluding bots) won't be following more than a few thousand followers maximum, it only takes a few hours to extract all the required data for a user. I wrote a simple script (here) to:
- Given a single user ID, get all friend IDs (accounts followed by the user)
- For each friend ID, get all *their* friend IDs 
- Dump the results into a json file.

After a few hours, I had all my data and was ready to start putting it together.

### Building and visualising a simple network

There are a number of different methods for building a graph network. At the most complex end we could put our data into an actual graph database like Neo4J. However for a graph of only a few thousand nodes we don't need to set up a whole database, we can simply load all the data straight from our json file into memory and use [NetworkX](https://networkx.org/), a Python for creating and analysing graph data structures. Given a list of users with a list of friend IDs for each user, we can:
- create a node for every user
- iterate through each user and for each of their friend IDs 
- if the friend ID exists as a node in our network, create an edge between the two. Here's some simple code to implement this with NetworkX.

```python
example_network_data = {
    '1': {
        'id': 1,
        'name': 'Tony',
        'friend_ids': [2, 3, 4, 11, 13, 14]
        },
    '2': {
        'id': 2,
        'name': 'Carmela',
        'friend_ids': [1, 4, 5, 6, 23, 25]
        },
    '3': {
        'id': 3,
        'name': 'Phil',
        'friend_ids': [1, 5, 35, 36, 39]
        }
}

def create_nx_network(network_data):

    # Initialise un-directional graph object
    graph = nx.Graph()
    
    # Create Nodes
    for user in tqdm(network_data.values()):
        graph.add_node(
            user['name'],
            user_id=user['id'])

    # Identify connections between user nodes    
    for user in tqdm(network_data.values()):

        for friend_id in user['friend_ids']:
            if friend_id in network_data.keys():

                source, target = user, network_data[friend_id]
                graph.add_edge(source['name'], target['name'])

    return graph

create_nx_network(example_network_data)
```

Of course, we also need to be able to visualise this graph. NetworkX has some basic functionality for drawing graphs with matplotlib, but for proper visualisation the best option is to export our graph into a readable file format and use a proper graph visualisation tool. I opted to use [Gephi] (https://gephi.org/), which allows us to customise how our network looks and also clustering/community-detection functionality (which we'll get into later!). Putting all of this together, lets started small by looking at my public twitter account - I don't use it much and only follow accounts people I already know in person, so it only follows a paltry *???*. Here it is visualised in Gephi. 

<img src="{{ site.url }}{{ site.baseurl }}/images/twitter_graphs/seitaro_unweighted.png" alt="JoeSeitaro Undirected & Unweighted Network">

The layout of the graph is determined with a [force-directed algorithm](Force-directed) (I've used *MultiGravity Force Atlas 2* implemented in Gephi), which assigns attractive, spring-like forces between two nodes with an edge and repulsive forces between all nodes in order to draw the graph in 2-dimensional space. As the central 'ego-node' with an edge to every other node my user account occupies the centre, and we can see right away without having to use any community-detection algorithms that the force-directed algorithm appears to naturally organise the nodes into three clusters. This based on the relative abundance of edges between nodes *within* a cluster that attracts the nodes together, and relative dearth of edges between nodes *across* different clusters that repels the clusters from each other (this is the basic principle of most community-detection algorithms, although purely for identifying groups rather than determining graph layout).

As it happens the clusters here represent three obvious real-world communities from my perspective. The bottom-left cluster are schoolfriends, the top-left cluster are friends from my undergraduate degree and the right cluster from my graduate degree. There are a few edges between the clusters (including a parody account, "@TheRealJoeWhite" set up in my honour by my university friends that my schoolfriends subsequently discovered) but otherwise the clusters are quite obviously distinct from each other. 

# Edge directions and Edge weights

[REWORD ALL THIS]For a small and simple network like this, this type graph representation is good enough for analysis. However it is missing two features that are important for analysing larger and more complex graphs. 


The first thing we need to consider is edge direction *direction*. The graph above is an un-directional graph, meaning that an edge between Node A and Node B simply represents a link between the two and nothing more. However, as alluded to in the earlier 'Getting Data out of Twitter' section however, edges on Twitter's social network all have a direction. Account A follows Account B which represents an outgoing edge from A to B - B may or may not follow back, and if they do there will be *two* edges between A and B, one going each way. User A may also be followed by a Account C, which represents an incoming edge to A from C. This makes Twitter's social network a [directed graph](https://www.wikiwand.com/en/Directed_graph), or digraph which we need to include in our model. Luckily, NetworkX and Gephi allow us to do just that.

The second crucial thing that we should consider is edge *weight*. This is the stuff that I'm really primarily interested in and so needs some proper explanation. In graph theory, the preceding graph is an unweighted graph where all edges are equivalent. A [weighted graph](https://www.wikiwand.com/en/Graph_(discrete_mathematics)#/Weighted_graph) in contrast assigns a separate weight value to each edge. These edge weights can represent all sorts of things depending on the graph context - for example the cost of travel between nodes or edge length but here we are going to consider edge weight as a measure of the *significance* of any given relationship between two accounts in a social network. The problem is the follower/friend data we've extracted from Twitter is unweighted, which means we need to decide for ourselves what significance actually means in our social network and the methods for measuring/calculating.

The social significance of an edge is a tricky concept so lets consider a real-world example. Your personal social network might include family members, colleagues at work and the cashier at your local corner. The relationships you have with these different people are (probably) not all of equal importance - some have more significance to both you and them than others. This relationship significance is what we want to represent to some degree through edge weighting in our ego-network because it is information that feeds into our analysis, both for our force-directed graph layout (greater edge weights are represented by greater attractive forces between nodes, drawing them closer together) and for the community detection algorithms we use (greater edge weights will be taken into account when identifying clusters/communities).

[...something on sociology...]

One way of assigning edge weight is to look at activity/interaction between accounts, which requires extracting extra data from twitter on how users are favouriting, retweeting or replying to each other's tweets - the idea being that more interaction between two users indicates a more significant relationship, which should be accordingly represented by greater edge weight. This *interaction-based* approach is demonstrated in [???] article referenced in the introduction - ??? introduces a formula for measuring the interaction between two users, which is based on ...

Here however I've taken a different approach to edge-weighting - partially because extracting all that extra interaction data out of twitter requires a lot of extra work and time! My main reason however is that I think ???'s interaction-based equation, while clearly being effective as showcased by his results, has a drawback in that it assumes everyone is using Twitter in the same way, and that it is therefore possible to say for example that retweeting is always x times more significant than favouriting. From my experience however people use Twitter in different ways - some will almost never retweet, while others will favourite and retweet interchangeably. While I could probably take a reasonable stab at manually assigning a weight for each type of interaction parameter, it's inevitably going to introduce some bias into the graph on the basis based on assumptions I make about users actually interact with Twitter. 

I've therefore stuck to using the follower/friend data I originally extracted. I think it's an interesting limitation in its own right to see how much can be done with this information alone. Without data on Twitter interaction and activity, what do we have? Well we have some basic information about each user - crucially, we have their follower count (how many accounts follow them) and their friends count (how many accounts they follow). From this, we can get two important things. 

From the follower count, we can get how "big/famous" an account is. User A with 3,000,000 followers is more famous than User B with just 50 followers. How could this be used to model edge weight? One idea is to think about edge weight as representation relationship "closeness" (???) a relationship is. If I follow User A, it is unlikely this represents a significant relationship. If they are a prominent author, musician or celebrity, I might know of them and even like their work, but there is unlikely to be a significant relationship between us - I am just one among 3 million others that have made the decision to follow this famous person. In contrast, lets say I follow User B. Given that so few accounts have seen and decided to follow this account, *my* personal decision to do so is relatively unique and interesting - it is likely that the social relationship represented by me following User B is more significant than the one represented by me following User A from the perspective of modelling community structure in our social network. I might have found or deliberately sought out their account because they were someone I already knew offline, or from a different social media platform. If they did pop up randomly in my feed (again, the smaller the account, the less likely this is to happen), I might follow them because of a commonality - perhaps we support the same football team, or have the same niche taste in music. From this idea, we can derive a basic principle for our model - *In a follower/followee relationship, the significance of the relationship is inversely correlated with the total number of followers the followee account has.*


From the friends count we can get something less obvious - a measure of how selective a user is in who they follow. User A with 2000 friends, who tends to follow back anyone who follows them, is less selective (and probably more active on Twitter, but we'll come to that later) than User B keeps their friend list trimmed to around 200 and regularly unfollows accounts they have previously followed if they decide they are no longer interested in them (this is how I personally use Twitter). Another way of thinking about this is in terms of how much *attention* a user can give to each of their friends - User A's attention is split between 2000 accounts while User B's attention is split between just 200. To put it in extremely crude mathematical terms, assuming User A and B spend an equal amount of time on Twitter (which they almost certainly do not!), as User B only follows 1/10th of accounts that User A does, they pay 10 times more attention to each of the accounts they follow. From this, we can derive our second basic principle for our model- *In a follower/followee relationship, the significance of the relationship is inversely correlated with the total number of friends the follower account has.*


So, we have our two basic principles:
- *In a follower/followee relationship, the significance of the relationship is inversely correlated with the total number of followers the followee account has.*
- *In a follower/followee relationship, the significance of the relationship is inversely correlated with the total number of friends the follower account has.*

Lets take these and combine them into an equation that we can use for calculating our edge-weights. Here's what I came up with.

$$ w = \frac{1}{f(a)}  \times \frac{1}{f(b)} $$
Where:
- $a$ represents the friend count for User A
- $b$ represents the follower count for User B]
- $f()$ represents a logarithmic growth function (which I'll come back to)



[BREAKDOWN_FROM_TEAMS]







[CAVEAT/FOOTNOTE]
Rather than looking at interaction, I've decided to focus heavily on a different aspect of Shaham's equation that he mentions. To quote:
    *"A) if a user has a low following-to-follower ratio, it makes their following someone more ‘significant’ and B) Users with more followers following users with less followers is more ‘significant’ than the other way around."*

This is a really interesting idea and worth exploring in more depth. It suggests for example that the account of a 'famous' user A with 2 million followers and only 500 following will always be a more 'significant' follower than user B with only 100 followers and 2000 following, which feels right. 






# Community detection and relationship strength


This is of course where things get tricky and to a certain extent subjective because while the existence of a link by itself is ??? (either I follow someone or I don't)
### Explanation of weighting - theory
    And implementation with graph
### Explanation of results with Sian's twitter
### Conclusion and what might come next