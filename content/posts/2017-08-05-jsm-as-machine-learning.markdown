---
title: "JSM as Machine Learning"
date: 2017-08-05T00:03:50+03:00
---


For a long time I wanted to establish a layman's description of JSM method, its application in machine learning and showcase a practical example of its use. Instead of focusing on the intricate theoretical foundation it will be based off the simplified definition obtained by [Anshakov](http://www.raai.org/about/persons/anshakov/ansh2012tmojsm.pdf "Set Theoretic Definition of JSM"). The paper is sadly in Russian as is the case with most information on JSM, hence today's post.

## Foundations

Let's assume we have N objects, each described by M binary attributes and a singular property which is one of "negative", "positive" or "tau" denoting the absence or the presence of property and tau being yet to be determined value. This set of objects is called the Context. Observe that we can map an arbitrary set of objects to a set of attributes
 common to all of them. Getting attributes that are common to a number of objects is an intersection of objects. Likewise we can pick an arbitrary set of attributes and find all objects that have these attributes. 
![An example context](/assets/images/Context.jpg "An example context")
When things get more interesting is when we combine both operations going e.g. attributes -> objects -> attributes or objects -> attributes -> objects. Observe that for instance we may start with a single attribute but once we intersect all objects having it we may obtain more attributes common to all of them. This operation (application of both tr
ansitions) is called closing the set of attributes or objects. On the figure above closing the 1st attribute gives us a set of 1st and 3rd attribute. Let me point out without 
proof that once a set is obtained by closure continuous application of closing operator is idempotent. The set is called closed set. Also a closed set can be described in any of two ways - a set of objects or corresponding set of attributes. Such a pair of equivalent sets of objects and attributes is called a concept. The concept is a crucial notion, as it represents what JSM calls a hypothesis provided however that all of objects have the same property. In this case it is understood that attributes are the cause of the property (or the absence of it) and objects are examples that support this hypothesis. Obtaining these hypotheses represent a typical empiric reasoning - induction. It is therefore prudent to mine all hypotheses from the context and latter apply the best of them for classification. The "best of them" is the important bit that I will touch on later.

## Basic JSM

First let's focus on the mining all concepts problem. Straightforward solution is  to intersect pairs of objects, then close the results, following that add a new object to each set and repeat. The problem is checking already obtained sets would require a large lookup table of sorts. An intriguing algorithm due to [Kuznetcov](https://www.hse.ru/data/2013/02/20/1306839179/Comparing%20performance%20of%20algorithms%20for%20generating%20concept.pdf "Comparing algorithms") is able to detect if the set was already obtained based on canonicalization rule without any lookups. Careful reader will observe that linked paper defines similar problem calling it formal concept analysis. Indeed the two branches of science are closely related, and easily borrow algorithms from one another.
Not going to describe the Close by One algorithm as its author outlines it better. Still consider that it essentially starts as our naive version but with attributes, adding them one by one and performing a closure after each addition. Importantly it discards a concept if after closure it gets any new attribute preceding the current one and not in the parent set. 

This is what makes close by one produce each unique concept (hypothesis) only once. Performing close by one on a set of all positive examples gives us positive hypotheses and likewise we can separately obtain all negative hypotheses. Then JSM procedure canceles out equal hypotheses of opposite sets and all hypotheses that fully intersect with some example of the opposite sign. Hypotheses thus obtained could be used as rules for classification. I would also point out without going into details that there are different strategies for filtering hypotheses beyond this one. 

Classification of tau examples is done by intersecting with hypotheses, each full intersection (if an intersection is equal to hypothesis) is considered as a vote in favor of (not) having a property. Here is where some tuning comes in - for a classic version only unanimous vote is considered as definitive. Non-classic JSM may choose to go with different majority proportions. Inconclusive votes leave example in a contradictory state, which is in contrast with most machine learning techniques that would give only 1 or 0 regardless. Moreover the whole set of posible answers is 4-fold: unknown (no covered by any hypothesis), positive, negative and contradiction. A lot of contradictions or unknowns means that one needs to work on the quality of training data and/or selection of attributes, also to consider non-classic JSM.

## Generalizing

Up to this point the problem space was fairly limited - only binary attributes and a single binary property. First an obvious extension is considering P independent properties at once. One can go about it by performing 2P separate inductions - each property times plus and minus. More interestingly it can be done in exactly one induction by encoding properties as a pair of attributes - one for minus and one for plus. Then a single pass of induction is performed discarding concepts whose parents properties have zero in intersection. Going to tau examples (which can be partially tau) during classification they count as two ones to allow simpler processing.

Getting more ambitious and speaking of non-binary attributes, a typical enumeration can be encoded as binary number. For continuous intervals there are more clever encodings. I will cover one particular encoding - "activity level", which is a number in (0, 1.0) range where having large value implies greater activity. The key observation is that activity of 0.5 implies having more then 0.25, therefore an intersection of activities is the minimum of the two. Finally the binary encoding is done by separating the range in a fixed number of steps and denoting a binary attribute for each step. With such an arrangement binary intersection will magically obtain the required logical result. This trick works both for properties and attributes. Other clever encodings are a topic for a separate article.
![Level encoding](/assets/images/LevelEncoding.jpg "Level encoding")

## Case study - Mushrooms

In this chapter I'm finally getting to something working in the real life, showcasing my Scala [library for JSM](https://github.com/DmitryOlshansky/jsm4s "JSM4S"). I also have a faster but more cumbersome to use version for C++ that was used to explore high performance parallel JSM induction algorithms and to pick the best data-structures. For a test drive I'm picking the ever so popular [mushroom data set](https://archive.ics.uci.edu/ml/datasets/mushroom) from UCI.

A few words sbout jsm4s - it's meant to be a flexible library with all manner of tuning but with simple comand-line tool that works out of the box. The cli tool picks non-classic JSM with majority vote predictor and reject by counter-example falsification strategy. As far as encoding goes it detects simple enumerated and numeric fields. For numerics it does a best-effort greedy algorithm to split the value range so that for each piece  there is some dominating property value.

For simplicity I'll start with command-line driver, the library itself will allow more flexibility that we do not require. First let's encode CSV to jsm4s native format:
```
java -jar jsm4s.jar encode -p 0 mushroom.csv mushroom.dat
```
0 indicates that the first column is the desired property. Next randomly split the dataset into training and validation, as 70%/30%:
```
java -jar jsm4s.jar split 7:3 mushroom.dat training.dat validation.dat
```
And make a tau set out of validation to run the classifier on:
```
java -jar jsm4s.jar tau validation.dat tau.dat
```
Now the busy work is done, on to the machine learning, obtain the model (hupotheses):
```
java -jar jsm4s.jar generate -m hypotheses.dat training.dat
```
Finally run the classifier:
```
java -jar jsm4s.jar predict -m hypotheses.dat -o output.dat tau.dat 
```
The last two steps could be combined, keeping the model in memory:
```
java -jar jsm4s.jar jsm -o output.dat training.dat tau.dat
```
Okay, time to analyze the results. jsm4s  calculates simple metric of correct predictions, conflicts and unknowns.
```
java -jar jsm4s.jar stats validation.dat output.dat 
```

There is also another [dataset](https://archive.ics.uci.edu/ml/datasets/adult) that I've successfully tested with jsm4s, though it takes a fair amount of time. I hope others might find JSM interesting and will try to apply it to wider range of data. In the age of Deep Learning dominance I think it's doubly important to keep obscure methods going.

