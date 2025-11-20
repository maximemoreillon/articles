+++
date = '2022-12-19'
title = "Mongoose query documents with matching array element"
tags = ['MongoDB', 'Mongoose', 'Tutorials']
+++

As a NoSQL database, MongoDB can store arrays as fields of a document. This article presents how to query such documents by filtering those with arrays that contain a specific value.

Let's imagine a collection containing the following movie records:

```
[
  {
    title: 'Inception',
    actors: [
      'Tom Hardy'
      'Leonardo DiCaprio'
    ]
  },
  {
    title: 'The Dark Knight Rises',
    actors: [
      'Tom Hardy',
      'Christian Bale',
    ]
  },
  {
    title: 'The Dark Knight',
    actors: [
      'Christian Bale',
      'Heath Ledger'
    ]
  },	
]
```

There would be cases where one would want to query all movies in which a specific actor starred.

With Mongoose, this can be achieved using the find() function as follows:

```
const moviesStarringTomHardy = await Movies.find({actors: 'Tom Hardy'})
```