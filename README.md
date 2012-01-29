recommendify
============

Incremental and distributed item-based "Collaborative Filtering" for binary ratings with ruby and redis. In a nutshell: You feed in `user -> item` interactions and it spits out similarity vectors between items ("related items"). 


### use cases

+ "Users that bought this product also bought...". 
+ "Users that viewed this video also viewed...". 
+ "Users that follow this person also follow...". 

etc.



### how it works

Recommendify keeps an incrementally updated `item x item` matrix, the "co-concurrency matrix". This matrix stores the number of times that a combination of two items has appeared in an interaction/preferrence set. The co-concurrence counts are processed with a similarity measure to retrieve another `item x item` similarity matrix, which is used to find the N most similar items for each item. This approach was described by Miranda, Alipio et al. [1]

1. Group the input user->item pairs by user-id and store them into interaction sets
2. For each item<->item combination in the interaction set increment the respective element in the co-concurrence matrix
3. For each item<->item combination in the co-concurrence matrix calculate the jaccard similarity
3. For each item store the N most similar items in the respective output set.


Fnord is not a draft!



usage
-----

You can add new interaction-sets to the processor incrementally, but the similarity matrix has to be manually re-processed after new interactions were added to any of the applied processors. However, the processing happens on-line and you can keep track of the changed items so you only have to re-calculate the changed rows of the matrix.

```ruby

# Our similarity matrix, we calculate the similarity via co-concurrence 
# of items in "orders" and the co-concurrence of items in user-likes using 
# two `item x item` matrices and the jaccard similarity measure.
class RecommendedItem < Recommendify::SimilarityMatrix

  # store a maximum of fifty neighbors per item
  max_neighbors 50

  processor :order_item_sim, 
    :similarity_func => :jaccard
    :weight => 5.0
  
  processor :like_item_sim,
    :similarity_func => :jaccard
    :weight => 1.0

end

# add `order_id->item_id` interactions to the processor (incremental)
order_item_processor = RecommendedItem.order_item_sim
order_item_processor.add_set("order1", ["item23", "item65", "item23"])
order_item_processor.add_set("order2", ["item14", "item23"])

# add `user_id->item_id` interactions to the processor (incremental)
like_item_processor = RecommendedItem.like_item_sim
like_item_processor.add_set("user1", ["item23", "item65", "item23"])
like_item_processor.add_set("user2", ["item14", "item23"])

# Calculate all elements of the similarity matrix
RecommendedItem.process!

# ...or calculate a specific row of the similarity matrix (a specific item)
RecommendedItem.process_item!("item65")

# retrieve similar items to "item23"
RecommendedItem.for("item23") 
  => [ <RecommendedItem item_id:"item65" similarity:0.23>, (...) ]

```


does it scale?
--------------

The maximum number of entries in the co-concurrence and similarity matrix is k(n) = (n^2)-(n/2), it grows O(n^2) and and would result in a maximum of 2000001 million entries for 2 million items. However, in a real scenario it is very unlikely that all item<->item combinations appear in a interaction set and we use a sparse matrix which will only use memory for elemtens with a value > 0.

The size of the computed nearest neighbors grows O(n). If we compute e.g. a maximum of 30 neighbors per item and assume no item_id is longer than 50 chars, then no set should be bigger than 50 + (50 * 30) = 2250bytes for the ids + (50 * 32) = 1600bytes for the scores, a total of 3850bytes + 25% redis overhead = 4812bytes per set. 

This means 2 million items will - in the worst case - require 2000000 * 2 * 4812bytes = 18,3 gigabyte of memory for the output data.


ideas
-----

+ rake benchmark CLASS=MySimilarityProcessor
+ NativeJaccardProcessor
+ CosineProcessor


Sources / References
--------------------

[1] Miranda C. and Alipio J. (2008). Incremental collaborative ﬁltering for binary ratings (LIAAD - INESC Porto, University of Porto)

[2] George Karypis (2000) Evaluation of Item-Based Top-N Recommendation Algorithms (University of Minnesota, Department of Computer Science / Army HPC Research Center)

[3] Shiwei Z., Junjie W. Hui X. and Guoping X. (2011) Scaling up top-K cosine similarity search (Data & Knowledge Engineering 70)



License
-------

Copyright (c) 2011 Paul Asmuth

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to use, copy and modify copies of the Software, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.