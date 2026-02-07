### Indexes are Models
The fundamental premise of this research is that traditional data structures are actually **models** that map a lookup key to a position or existence of a record.
*   **B-Trees:** Regression models that predict the position of a key in a sorted array.
*   **Hash-maps:** Models that map a key to a position in an unsorted array.
*   **Bloom Filters:** Binary classifiers that predict if a key exists in a set.

### Theoretical Foundation
For sorted data, an index is effectively an approximation of the **Cumulative Distribution Function (CDF)**. 
*  Predicted Position $p = F(Key) \times N$, where $F(Key)$ is the estimated percentile of the key and $N$ is the total record count.
*  Traditional B-Trees scale at $O(\log n)$ for search and $O(n)$ for index memory. By using a mathematical function to calculate the position, learned indexes can theoretically achieve **$O(1)$ lookup time and $O(1)$ index memory**.

### The Recursive Model Index Architecture (RMI)
A single model often struggles with "last-mile" accuracy (the precision needed to find a specific record among millions). The RMI solves this via a **hierarchy of experts**:
*   **Stage-wise Refinement:** A top-level model predicts the general region and selects a specialized "expert" model in the next stage.
*   **No Inter-stage Search:** Unlike B-Trees, there is no searching between levels; the output of one model is used as a direct mathematical address to the next.
*   **Hybrid Guarantees:** These systems can be "hybrid," automatically replacing poor-performing ML models at the leaf level with traditional B-Trees to guarantee that worst-case performance never falls below classical standards.

### Hardware Evolution: Logic vs. Math
The transition to learned indexes aligns with modern hardware trends:
*   **CPU Stagnation:** Classical indexes rely on **branch-heavy logic** (if-statements), which is increasingly difficult for modern CPUs to parallelize.
*   **Mathematical Throughput:** Modern hardware (SIMD, GPUs, TPUs) is optimized for **parallel arithmetic**. Learned indexes turn the search problem into a calculation, allowing systems to benefit from the massive growth in floating-point and integer math performance.

### Point and Existence Indexes
*   **Learned Hashing:** By using the CDF as a hash function, models can learn the data distribution to significantly reduce conflicts compared to randomized hashing.
*   **Learned Bloom Filters:** These frame existence as a classification task. To maintain the "no false negatives" guarantee, they use a model to filter easy cases and a small **overflow Bloom filter** for cases where the model is uncertain.