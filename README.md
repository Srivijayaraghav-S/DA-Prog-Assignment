# **DA-Assignment**

## by Srivijayaraghav Srinivasan, 106121131

# **Implementing SON and Toivonen's Algorithm Using MapReduce**
The SON algorithm and the Toivonen algorithm are both widely used approaches for solving the problem of frequent itemset mining in large datasets. These algorithms are particularly well-suited to distributed computing environments, where data is too large to fit into the memory of a single machine.

Implementing these algorithms in MapReduce or Apache Spark leverages the power of distributed computing to handle very large datasets efficiently. In a MapReduce context, the mapper function can distribute data partitions across nodes for local frequent itemset mining (Phase 1 of SON or the sampling step of Toivonen), while the reducer function aggregates the results across all nodes for the global verification step. Apache Spark, with its in-memory computing capabilities, offers a more efficient and faster platform for these algorithms, especially due to its optimization for iterative algorithms like Apriori, which is used within both SON and Toivonen algorithms, while Spark's resilient distributed datasets (RDDs) and dataframes provide flexible abstractions for distributing data and computations.

## **SON Algorithm**
The SON algorithm, proposed by Savasere, Omiecinski, and Navathe, breaks down the task of identifying frequent itemsets into two phases to make it manageable across distributed systems. In the first phase, the algorithm partitions the dataset and applies the Apriori algorithm to each partition to find local frequent itemsets. This phase significantly reduces the dataset's size that each node must handle, allowing the algorithm to scale efficiently with data size. In the second phase, the algorithm aggregates the local frequent itemsets from all partitions and then scans the entire dataset to determine which of these itemsets are indeed frequent across the whole dataset. The SON algorithm's is simple and effective, enabling parallel processing without missing any frequent itemsets.

## **Working of SON Algorithm**

## First Pass
*   Repeatedly read small subsets of the baskets into main memory
*   Run an in-memory algorithm (e.g., Apriori, random sampling) to find all frequent itemsets\
(Note: we are not sampling, but processing the entire file in memory-sized chunks)
*   An itemset becomes a candidate if it is found to be frequent in any one or more subsets of the baskets

## Second Pass
*   Count all the candidate itemsets and determine which are frequent in the entire set
*   Key **“monotonicity”** idea: an itemset cannot be frequent in the entire set of baskets unless it is frequent in at least one subset
*   Subset or chunk contains fraction *p* of whole file
*   *1/p* chunks in file
*   If itemset is not frequent in any chunk, then support in each chunk is less than *ps*
*   Support in whole file is less than *s*: not frequent

## **SON: MapReduce**
## Phase 1: Find Candidate Itemsets
**Map:**
*   Input is a chunk/subset of all baskets - fraction *p* of total input file
*   Find itemsets frequent in that subset (e.g., using Apriori algorithm)
*   Use support threshold *ps*
*   Output is set of key-value pairs (*F*, 1), where *F* is a frequent itemset from sample

**Reduce:**
*   Each reduce task is assigned set of keys, which are itemsets
*   Produces keys that appear one or more time
*   Frequent in some subset
*   These are candidate itemsets

## Phase 2: Find True Frequent Itemsets
**Map:**
*   Each Map task takes output from first Reduce task AND a chunk of the total input data file
*   All candidate itemsets go to every Map task
*   Count occurrences of each candidate itemset among the baskets in the input chunk
*   Output is set of key-value pairs (*C*, *v*), where *C* is a candidate frequent itemset and *v* is the support for that itemset among the baskets in the input chunk

**Reduce:**
*   Each reduce tasks is assigned a set of keys (itemsets)
*   Sums associated values for each key: total support for itemset
*   If support of itemset >= *s*, print itemset and its count

However, even with SON algorithm, we still don’t know whether we found all the frequent itemsets, as an itemset may be infrequent in all subsets but frequent overall - Toivonen's algorithm solves this.

## **Toivonen's Algorithm**
The Toivonen algorithm introduces a probabilistic approach to frequent itemset mining. It starts by selecting a random sample of the dataset and then applies the Apriori algorithm to this sample to find potential frequent itemsets and generate a "negative border" – itemsets that are not frequent in the sample but are close to the threshold. The entire dataset is then scanned to verify which of the sampled frequent itemsets are genuinely frequent and to ensure that no itemsets in the negative border are frequent. This method reduces the computational cost by potentially requiring only one full scan of the dataset, at the cost of having to handle the complexity of dealing with the negative border.

## **Working of Toivonen's Algorithm**

## First Pass
Find candidate frequent itemsets from sample
*   *Use lower threshold* : For fraction *p* of baskets in sample, use *0.8ps* or *0.9ps* as support threshold - identifies itemsets that are frequent for the sample
*   Construct the *negative border* - itemsets that are not frequent in the sample but all of their immediate subsets are frequent

## Second Pass
Process the whole file (no sampling)
*   Count all candidate frequent itemsets from the first pass and all itemsets on the negative border
*   *Case 1* : No itemset from the negative border turns out to be frequent in the whole data set - correct set of frequent itemsets is exactly the itemsets from the sample that were found frequent in the whole data
*   *Case 2* : Some member of negative border is frequent in the whole data set - can give no answer at this time and must repeat the algorithm with a new random sample

## **Why Toivonen's Algorithm Works**
Toivonen’s algorithm never constructs a false positive, since it only describes as frequent those itemsets that have been counted and found to be frequent in the total. It also never constructs a false negative, as when no itemset of the negative border is frequent in the whole, there can be no itemset that is both frequent in the complete itemset and present in neither the negative border nor the collection of frequent itemsets for the given sample.
