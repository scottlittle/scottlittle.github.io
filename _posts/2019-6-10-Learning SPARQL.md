---
layout: post
title: Learning SPARQL Query Language
---

**Disclaimer**: I'm a SPARQL and semantic web beginner.

SPARQL graph query language takes some getting used to for querying compared to normal SQL.  The promise is that you should be able to make more powerful queries quicker, diving into insights not as obvious with "normal" querying methods.  One drawback that I see is that it takes quite a bit of effort to get set up.  The data is very structured going into the graph database. This is the price to pay up front for the big promises of speed and scalability.  I believe that the package neo4j offers the ability to query using regular SQL, but if you do that, you might miss out on the insights and "ease" of a similar query in the SPARQL query language.

What good is all of this?  There is the WikiData efforts, coming out of the FreeBase project (in which Google bought).  WikiData allows one to make quick and powerful queries on the data of WikiData.  From a natural language processing (NLP) point of view, this is a lot different than how a data scientist might ingest the data of Wikipedia- that is, word2vec, doc2vec, tf-idf on the articles and words.  Instead of operating on English words (or any language for that matter), the graph database makes use of semantic concepts that can be queried and turned into English or another language as an afterthought (almost).

If I update this, I'll show some basic examples here.
