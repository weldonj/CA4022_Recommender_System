## BACKGROUND AND APPROACHES

**Background -** 

One of the big reasons that music streaming giant Spotify has been and continues to be so successful is, is its “Discover Playlist” feature. Premium users receive a unique playlist each week that Spotify has generated just for them, based solely on their own music listening habits. Anecdotally, through our own use of Spotify we have seen that these can be astonishingly accurate.

With this in mind, we are looking forward to attempting to create a music recommendation system that does some similar things. There are two main approaches that are used to build these systems.

**Collaborative Filtering -**

This approach to recommendation requires a lot of data about users’ preferences. In our case, with the last.fm dataset, it will be the weighting each user gives to music artists. Collaborative filtering doesn’t know anything about the artists themselves, it doesn’t attempt to analyse them at all. Instead it makes the assumption that if User A tends to give similar weights to music artists that User B, they both enjoy similar music. Therefore, if User B likes an artist that User A has not been exposed to yet, it’s likely that User A will be happy to have that artist recommended to them. This user preference data can either be explicit (the user actually rates the artists) or implicit (analyse which artists the user listens to most often and for long periods of time). Due to Collaborative Filtering’s reliance on user data, it suffers from what’s known as the “Cold Start” problem. Brand new users have no preference data, and so the system will always struggle to know what to recommend.

**Content-Based Filtering -**

Unlike the previous approach, here the items being recommended are also closely analysed. It can be a lot more difficult to collect the data as it’s not at all easy to identify how the individual features relate to user preferences. Two main data categories are used in content based music recommendation systems. Metadata about the songs or artists, and also the audio content itself. This can be features such as beats per minute, what chords are played in the song, and even just how loud it is. One big advantage that content based filtering has over collaborative filtering is that it can be difficult to get users to actually rate music, whereas songs and artists can be freely analysed.

**Hybrid Model -**

This is exactly as it sounds, with a blend of both collaborative filtering and content based filtering. These models can outperform either of the other two on their own, but it is difficult to find data that combines both user music ratings and actual data about the audio content itself.

**Sections -**

 - Data preparation

 - Matrix Factorization Recommender System

 - Spotify Integration

 - TSNE Clustering

 - Graph Analysis

 - Concluding Section