# FAISS & Similarity Search

**how** to find similar items in massive datasets without checking every single item one by one.

## What is Similarity Search?

Comparing query to all 1 million image is slow.

 FAISS organize data so we only check small, likely subset of images.

### The Vector Representation
Computers don't see "image" or "word"; they see **vectors** . 
-  sentence = `[0.1, 0.5, -0.2]`.
- image = `[0.9, 0.1, 0.4]`.

In FAISS, similar items are vectors that are "close" to each other in space.

```mermaid
graph LR
    A[Text/Image] --> B[Embedding Model]
    B --> C[Vector: 0.1, 0.5, 0.2]
    C --> D[FAISS Index]
    D --> E[Similarity Search]
```

---

## How do we measure Closeness?


### A. L2 (Euclidean) Distance
sensitive to both  **direction** and **magnitude** (length) of vector.

```mermaid
graph TD
    subgraph L2_Distance
    P1((Point A)) -- "Straight Line (L2)" --- P2((Point B))
    end
```

### B. Dot Product
measure how much two vectors point in same direction. It is sensitive to magnitude.
 

### C. Cosine Similarity
measure **angle** between vectors, ignoring their length. It normalize vectors first.
*   **Use when:** Text similarity (a long document and short sentence can be "similar" if they talk about same topic).

```mermaid
graph LR
    subgraph Cosine_Similarity
    Origin((0,0)) --> V1[Vector A]
    Origin --> V2[Vector B]
    V1 -- "Angle Theta" --- V2
    end
```

---

## 3. The Core Problem: ANN (Approximate Nearest Neighbor)


### \(\epsilon\)-tolerance Zone
accept vector close to nearest neighbor.

```mermaid
graph TD
    subgraph ANN_Concept
    Q((Query Vector)) -- "True Nearest Distance (d)" --- TrueNN[True Nearest Neighbor]
    Q -- "Acceptable Distance (1+ε)d" --- Acceptable[Acceptable Approximation]
    Q -- "Too Far" --- Reject[Rejected Vector]
    end
    style TrueNN fill:#4CAF50,color:white
    style Acceptable fill:#8BC34A,color:white
    style Reject fill:#F44336,color:white
```

*   **\(\epsilon = 0\)**: Exact search (Slow).
*   **\(\epsilon > 0\)**: Approximate search (Fast, but might miss absolute best match).

---

## 4. How FAISS Organizes Data

use different "Indexes" to speed up search.

### A. Flat Index 
store vectors exactly as they are and compare query to every single vector.
*   **Pros:** 100% Accurate.
*   **Cons:** Very slow for large datasets.
*   **Class:** `IndexFlatL2`

### B. IVF (Inverted File Index) - Clustering

1.  **Clustering:** space divided into clusters.
2.  **Search:** query only checks clusters closest to it, ignoring rest.

```mermaid
graph TD
    subgraph IVF_Search
    Q((Query)) --> C1[Cluster 1]
    Q --> C2[Cluster 2]
    Q -.-> C3[Cluster 3 - Ignored]
    Q -.-> C4[Cluster 4 - Ignored]
    C1 --> R[Search only these vectors]
    C2 --> R
    end
    style C3 fill:#ddd,stroke-dasharray: 5 5
    style C4 fill:#ddd,stroke-dasharray: 5 5
```
*   **Parameter `nprobe`:** How many clusters to check. Higher = More accurate but slower.
*   **Class:** `IndexIVFFlat`

### C. PQ (Product Quantization) - Compression

compress vectors to save memory.


```mermaid
graph LR
    subgraph PQ_Compression
    V[128-dim Vector] --> S1[Sub-vector 1]
    V --> S2[Sub-vector 2]
    V --> S3[Sub-vector 3]
    V --> S4[Sub-vector 4]
    S1 --> C1[Code: 42]
    S2 --> C2[Code: 187]
    S3 --> C3[Code: 5]
    S4 --> C4[Code: 221]
    C1 & C2 & C3 & C4 --> Final[PQ Code: 42, 187, 5, 221]
    end
```
*   **Class:** `IndexPQ`, `IndexIVFPQ`

### D. HNSW (Hierarchical Navigable Small World) - Graph

- **Top Layer:** Highways connecting distant cities.
- **Bottom Layer:** Local streets connecting neighbors.
- **Search:** Start at top, zoom down to specific neighborhood.

```mermaid
graph TD
    subgraph HNSW_Layers
    L2[Layer 2: Highways] --> L1[Layer 1: Major Roads]
    L1 --> L0[Layer 0: Local Streets]
    L0 --> Target((Target Vector))
    end
    style L2 fill:#FF9800,color:white
    style L1 fill:#FFC107,color:black
    style L0 fill:#4CAF50,color:white
```
*   **Parameter `M`:** Number of connections per node. Higher = More accurate, more memory.
*   **Class:** `IndexHNSWFlat`

---

## 5. Quantization: Saving Memory (SQ vs PQ)

Quantization reduce precision of numbers to save space.

### Scalar Quantization (SQ)

- **Original:** 32-bit float (0.123456)
- **Quantized:** 8-bit integer (42)

```mermaid
graph LR
    subgraph SQ_Process
    F[Float 32-bit] -->|Round| I[Integer 8-bit]
    I -->|Store| M[Memory Savings: 4x]
    end
```

### Product Quantization (PQ)
Splits vector into segments and quantizes each segment.


```mermaid
graph TD
    subgraph PQ_Steps
    Step1[Step 1: Original Vector D=128] --> Step2[Step 2: Partition into m=4 subvectors]
    Step2 --> Step3[Step 3: Match to Codebooks]
    Step3 --> Step4[Step 4: PQ Code - 4 bytes]
    end
    style Step4 fill:#2196F3,color:white
```

---

## 6. Choosing Right Index (Decision Tree)


```mermaid
graph TD
    Start((Start)) --> Size{Dataset Size?}
    Size -- "< 10k" --> Flat[IndexFlatL2<br>Exact Search]
    Size -- "> 10k" --> Exact{Need Exact?}
    
    Exact -- "Yes" --> FlatSlow[IndexFlat<br>Slow but Exact]
    Exact -- "No (ANN)" --> Memory{Memory Constrained?}
    
    Memory -- "Yes" --> IVF_PQ[IndexIVFPQ<br>Compressed]
    Memory -- "No" --> Speed{Speed Priority?}
    
    Speed -- "Accuracy" --> IVF_Flat[IndexIVFFlat<br>Good Balance]
    Speed -- "Speed" --> HNSW[IndexHNSW<br>Fastest Search]
```

---

## 7. Key FAISS Concepts Summary

| Concept | Description | Key Parameter |
| :--- | :--- | :--- |
| **Vector** | list of numbers representing object | Dimension (d) |
| **L2 Distance** | Straight-line distance | - |
| **Cosine** | Angle between vectors | - |
| **IVF** | Clustering to reduce search space. | `nprobe` |
| **PQ** | Compression by splitting vectors. | `m` (subvectors) |
| **HNSW** | Graph-based navigation (Highways). | `M`, `efSearch` |
| **SQ** | Simple compression (32-bit to 8-bit). | `QT_8bit` |

## 8. Practical Example

```python
import faiss
import numpy as np

# 1. Setup
d = 128          # Dimension
nb = 100000      # Database size
nq = 10          # Queries

# 2. Create Data
xb = np.random.random((nb, d)).astype('float32')
xq = np.random.random((nq, d)).astype('float32')

# 3. Choose Index (IVF + PQ for speed and memory)
nlist = 100      # Number of clusters
m = 8            # Number of subquantizers
quantizer = faiss.IndexFlatL2(d)
index = faiss.IndexIVFPQ(quantizer, d, nlist, m, 8)

# 4. Train (Learn the clusters and codebooks)
index.train(xb)

# 5. Add Data
index.add(xb)

# 6. Search
index.nprobe = 10  # Check 10 closest clusters
D, I = index.search(xq, 5) # Find 5 nearest neighbors

print("Distances:", D)
print("Indices:", I)
```

---

## 9. Vector Visualization (SVG)

Here is a visual representation of how vectors relate in space. 

<svg width="400" height="300" xmlns="http://www.w3.org/2000/svg">
  <!-- Grid -->
  <defs>
    <pattern id="grid" width="20" height="20" patternUnits="userSpaceOnUse">
      <path d="M 20 0 L 0 0 0 20" fill="none" stroke="#eee" stroke-width="1"/>
    </pattern>
  </defs>
  <rect width="100%" height="100%" fill="url(#grid)" />

  <!-- Axes -->
  <line x1="50" y1="250" x2="350" y2="250" stroke="black" stroke-width="2" />
  <line x1="50" y1="250" x2="50" y2="50" stroke="black" stroke-width="2" />
  <text x="360" y="265" font-family="Arial" font-size="12">Dimension 1</text>
  <text x="20" y="40" font-family="Arial" font-size="12">Dimension 2</text>

  <!-- Vector A -->
  <line x1="50" y1="250" x2="150" y2="150" stroke="#2196F3" stroke-width="3" marker-end="url(#arrowhead)" />
  <text x="160" y="145" fill="#2196F3" font-family="Arial" font-size="14" font-weight="bold">Vector A</text>

  <!-- Vector B -->
  <line x1="50" y1="250" x2="250" y2="100" stroke="#F44336" stroke-width="3" marker-end="url(#arrowhead)" />
  <text x="260" y="95" fill="#F44336" font-family="Arial" font-size="14" font-weight="bold">Vector B</text>

  <!-- Vector C (Similar to A) -->
  <line x1="50" y1="250" x2="130" y2="170" stroke="#4CAF50" stroke-width="3" marker-end="url(#arrowhead)" />
  <text x="140" y="180" fill="#4CAF50" font-family="Arial" font-size="14" font-weight="bold">Vector C</text>

  <!-- Angle Arc for Cosine -->
  <path d="M 80 220 Q 100 200 120 200" fill="none" stroke="#FF9800" stroke-width="2" />
  <text x="100" y="215" fill="#FF9800" font-family="Arial" font-size="12">θ</text>

  <!-- Arrowhead Marker -->
  <defs>
    <marker id="arrowhead" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
      <polygon points="0 0, 10 3.5, 0 7" fill="#333" />
    </marker>
  </defs>
</svg>

*   **Vector A (Blue)** and **Vector C (Green)** are close together. They have small angle (\(\theta\)) between them, meaning they are **similar** (high Cosine Similarity).
*   **Vector B (Red)** is far away. It points in different direction. It is **dissimilar**.
*   **L2 Distance** measure straight line between tips of Vector A and Vector C.
*   **Inner Product** measure how much Vector A "projects" onto Vector C.