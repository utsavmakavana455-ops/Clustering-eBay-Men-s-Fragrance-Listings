# Clustering eBay Men's Fragrance Listings


Unsupervised product segmentation of 1,000 eBay men's fragrance listings. The project groups
products by **price, units sold and stock level** using three clustering methods (K-Means, Ward
hierarchical clustering and DBSCAN), compares them, and turns the result into four business
segments with practical recommendations.



## 1. Project goal

An e-commerce company holds a large catalogue of products with prices, stock levels and sales
counts. The question: **which products behave alike, and what should the business do about each group?**

There is no "right answer" column, so this is an *unsupervised* problem. Classification measures
such as accuracy do not apply. Instead, the models are judged on:

- **Internal scores:** how tight and well separated the groups are.
- **Stability:** do the groups stay the same if the data or random seed changes?
- **Business use:** can a manager name each group and act on it?

## 2. Key results

Final model: **K-Means with 4 clusters**.

| Segment | Listings | Median price | Median units sold | Median stock | Share of est. sales value* |
|---|---|---|---|---|---|
| Core best-sellers | 249 (25.1%) | $34.84 | 550 | 10 | **78.4%** |
| High-volume, deep stock | 84 (8.5%) | $26.92 | 964.5 | 82.5 | 15.0% |
| Premium, slow sellers | 294 (29.7%) | $74.88 | 21 | 6 | 5.4% |
| Budget, slow sellers | 364 (36.7%) | $24.39 | 20 | 8 | 1.2% |

\*Estimated sales value = price × units sold. This is revenue, not profit.

**Main finding:** one quarter of listings produce about 78% of estimated sales value, while more
than a third produce about 1%.

| Model | Clusters | Noise % | Silhouette | Davies-Bouldin | Calinski-Harabasz |
|---|---|---|---|---|---|
| **K-Means (k = 4), selected** | 4 | 0.0 | 0.309 | 1.024 | 515.1 |
| Ward hierarchical (k = 4) | 4 | 0.0 | 0.268 | 1.096 | 429.9 |
| Ward hierarchical (k = 2) | 2 | 0.0 | 0.440 | 0.808 | 326.1 |
| DBSCAN (eps 0.6, min_samples 6) | 2 | 6.3 | 0.398 | 0.750 | 183.8 |

How to read the scores: silhouette and Calinski-Harabasz are better when **higher**;
Davies-Bouldin is better when **lower**.

## 3. Dataset

- **Source:** supplied by the university (`Sales-[5444].csv`), not included here.
- **Size:** 1,000 rows, 10 columns, no duplicate rows.
- **Content:** eBay men's fragrance listings captured on 22-24 May 2024: brand, title, category,
  price, stock, "sold" text, units sold, last-updated time and seller location.

Data problems found and how each was handled:

| Problem | Handling |
|---|---|
| `sold` column contradicts the listing text (for example "Last One / 6 sold" shows 7,654) | Units sold taken from the listing text; the column is only a fallback |
| 5 missing prices | Recovered from the displayed price string |
| 3 prices in Canadian dollars | Removed (no exchange rate given); 991 listings remain |
| 111 missing stock values (59 left after cleaning are "Limited quantity") | Parsed from text where possible; the rest set to the median (9) |
| 63 spellings of a few fragrance types | Merged into 4 groups: Eau de Toilette, Eau de Parfum, Cologne, Other |
| Very skewed numbers (units sold skewness 9.77; largest seller 54,052 units) | Log transform; outliers kept because they are real best-sellers |


## 4. Method

### 4.1 Features and preparation

- **Chosen features:** price, units sold, stock available. They describe positioning, demand and supply.
- **Left out:** free-text description, last-updated time (only 3 days), seller location (93% US) and
  brand name (246 values, many used once). Brand and fragrance type are used afterwards to *describe* clusters.
- **Transform:** `log(1 + x)` on all three, then standardise (mean 0, standard deviation 1).
- **Categorical test:** adding fragrance type as one-hot columns raised the K-Means silhouette at k = 4
  from 0.309 to 0.459. This was rejected: binary columns create artificially clean gaps, so the higher
  score reflected the encoding, not better groups.


### 4.2 Models

| Method | Why it was used | Settings and how they were chosen |
|---|---|---|
| **K-Means** | Fast, suits compact groups in standardised data | k = 2 to 10, 20 starts each, seed 42; k chosen from silhouette, Calinski-Harabasz and Davies-Bouldin (the elbow curve has no sharp bend) |
| **Ward hierarchical** | No fixed k needed; dendrogram shows structure; same objective as K-Means | Ward, average and complete linkage tested; Ward kept (average linkage's higher score came from clusters of 906, 82 and 3 listings) |
| **DBSCAN** | Finds oddly shaped groups and flags outliers | min_samples = 6 (2 × number of features); eps from the k-distance plot and a grid search over eps 0.2-1.0 and min_samples 4, 6, 10, 15 |
| **PCA** | Visualisation only | First two components explain 81.8% of the variance |

### 4.3 Evaluation

Silhouette, Davies-Bouldin and Calinski-Harabasz; adjusted Rand index (agreement) between models;
K-Means stability across 30 random starts and 30 resamples of 80% of the data; and a robustness
check on the imputed stock values.

## 5. Results in detail

### 5.1 Choosing k for K-Means

Silhouette (0.309) and Calinski-Harabasz (515.1) peak at k = 4, and Davies-Bouldin (1.024) is close to
its lowest value. A silhouette near 0.3 means the segments are real but overlap, which is typical of
retail data.


### 5.2 Hierarchical clustering


Cutting at four clusters gives silhouette 0.268. Two clusters score best (0.440) but are too coarse to guide decisions.

### 5.3 DBSCAN


With eps = 0.6, DBSCAN finds two clusters (880 and 49 listings) and 62 noise points (6.3%). The noise
points are high-volume listings (median 571 units sold against 45 for the rest), so DBSCAN works best
here as an outlier flag, not a segmenter. Settings that score higher, such as eps = 0.8, simply merge
nearly everything into one cluster of 970.

### 5.4 The models in PCA space


There are no empty gaps between groups: the data form a continuum from slow to fast sellers, which matches the modest silhouette.

### 5.5 Agreement and stability

| Check | Result |
|---|---|
| K-Means vs Ward (adjusted Rand index) | 0.571 (moderate agreement) |
| K-Means or Ward vs DBSCAN | 0.09 to 0.10 (largely unrelated) |
| K-Means across 30 random starts | mean 0.969, minimum 0.948 |
| K-Means across 30 resamples (80% of data) | mean 0.941, minimum 0.769 |
| Re-fit on the 932 listings with known stock | 0.882 agreement with main result |
| Re-fit without stock at all | 0.498 agreement (stock is what separates the high-volume group) |

### 5.6 Cluster profiles


- **Core best-sellers:** mid-priced, high volume; 63% Eau de Toilette; most sales value.
- **High-volume, deep stock:** cheap, very high volume, large stock; 71% Eau de Toilette; all US-based.
- **Premium, slow sellers:** highest prices but few sales; most Eau de Parfum (48%) and most non-US listings (15.6%); frequent brands include Giorgio Armani, Paco Rabanne and Yves Saint Laurent.
- **Budget, slow sellers:** cheapest and slow; holds 28 of the 39 "Unbranded" listings.

## 6. Business recommendations

1. **Protect the core best-sellers.** Running out of stock here is the costliest mistake. The platform only shows "more than 10 available", so check real stock.
2. **Review the high-volume, deep-stock group.** Low price plus large stock may tie up cash; check margins and holding costs.
3. **Test action on the premium, slow sellers.** Trial promotion of the best-known brands or a smaller range, and measure the effect on margin.
4. **Rationalise the budget, slow sellers.** Clearest candidate for clearance or delisting.
5. **Validate first.** Compare actual margins and weekly sales for 20 to 30 listings per segment with the profiles before making big changes.

## 7. Repository structure

```
.
├── COM7022_clustering.ipynb   # full analysis, executed, with outputs saved
├── data/                      # put Sales-[5444].csv here (not included)
│   └── README.md
├── figures/                   # the 8 figures saved by the notebook
├── results/                   # CSV tables
│   ├── kmeans_scores.csv      #   scores for k = 2 to 10
│   ├── dbscan_grid.csv        #   eps / min_samples search
│   ├── model_comparison.csv   #   side-by-side model scores
│   └── cluster_profiles.csv   #   profile of each K-Means cluster
├── report/                    # final write-up (Word and PDF)
├── requirements.txt
├── LICENSE
└── README.md
```

## 8. How to run

**Local**

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
# place the dataset at data/Sales-[5444].csv
jupyter notebook COM7022_clustering.ipynb
```

Run all cells from the top. The notebook creates `figures/` and `results/` if they are missing.

**Google Colab**

Upload the notebook and the CSV, then choose Runtime > Run all. The notebook looks for the file in
`data/`, the working folder, then `/content/`.

Results are reproducible: every random step uses seed 42.

**Tools:** Python, pandas, NumPy, scikit-learn, SciPy, matplotlib, seaborn.

## 9. Limitations

- Three-day snapshot from one marketplace, so seasonal and channel effects are unknown.
- Units sold are cumulative and listing age is missing, so older listings look stronger.
- 59 stock values are imputed (45 of them fall in the core best-sellers), and stock is capped by the "more than 10" display.
- A silhouette of about 0.31 means listings near a boundary could belong to a neighbouring segment.
- "Sales value" ignores cost, so it is not profit.
- The `sold` column contained at least one clear error.

## 10. Possible improvements

- Add listing age, cost and margin data.
- Repeat over several time periods to see whether the segments last.
- Extract size and concentration from the listing title text.
- Try a Gaussian mixture model, which gives each listing a soft membership across overlapping groups.

## 11. References

- Arthur, D. and Vassilvitskii, S. (2007) 'k-means++: the advantages of careful seeding', *Proceedings of the 18th Annual ACM-SIAM Symposium on Discrete Algorithms*, pp. 1027-1035.
- Caliński, T. and Harabasz, J. (1974) 'A dendrite method for cluster analysis', *Communications in Statistics*, 3(1), pp. 1-27.
- Davies, D.L. and Bouldin, D.W. (1979) 'A cluster separation measure', *IEEE Transactions on Pattern Analysis and Machine Intelligence*, PAMI-1(2), pp. 224-227.
- Ester, M., Kriegel, H.-P., Sander, J. and Xu, X. (1996) 'A density-based algorithm for discovering clusters in large spatial databases with noise', *KDD-96*, pp. 226-231.
- Hubert, L. and Arabie, P. (1985) 'Comparing partitions', *Journal of Classification*, 2(1), pp. 193-218.
- Jolliffe, I.T. and Cadima, J. (2016) 'Principal component analysis: a review and recent developments', *Philosophical Transactions of the Royal Society A*, 374(2065), 20150202.
- Lloyd, S.P. (1982) 'Least squares quantization in PCM', *IEEE Transactions on Information Theory*, 28(2), pp. 129-137.
- McKinney, W. (2010) 'Data structures for statistical computing in Python', *Proceedings of the 9th Python in Science Conference*, pp. 56-61.
- Pedregosa, F. et al. (2011) 'Scikit-learn: machine learning in Python', *Journal of Machine Learning Research*, 12, pp. 2825-2830.
- Rousseeuw, P.J. (1987) 'Silhouettes: a graphical aid to the interpretation and validation of cluster analysis', *Journal of Computational and Applied Mathematics*, 20, pp. 53-65.
- Schubert, E., Sander, J., Ester, M., Kriegel, H.-P. and Xu, X. (2017) 'DBSCAN revisited, revisited: why and how you should (still) use DBSCAN', *ACM Transactions on Database Systems*, 42(3), article 19.
- Ward, J.H. (1963) 'Hierarchical grouping to optimize an objective function', *Journal of the American Statistical Association*, 58(301), pp. 236-244.

