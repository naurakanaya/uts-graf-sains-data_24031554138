# UTS Graf Sains Data 2026/2027: Neo4j Python Driver and Recommendation

**Name:** Naura Kanaya Putri Masruri  
**NIM:** 24031554138  
**Class:** INT24
**Course:** Graf Sains Data, S1 Sains Data, Universitas Negeri Surabaya

An item-based KNN recommender built on the [RetailRocket e-commerce dataset](https://www.kaggle.com/datasets/retailrocket/ecommerce-dataset), using Neo4j as the graph store and the Neo4j Python driver for import, querying and similarity write-back.

## Repository contents

```
.
├── UTS_Graph_Naura_Kanaya_Putri_Masruri_24031554138.ipynb  
├── requirements.txt
├── data/
│   ├── events_filtered.csv   # P1 + P2 events (filtered by timestamp)
│   ├── events_p1.csv         # training period, imported into Neo4j
│   └── events_p2.csv         # testing period, used as ground truth
└── images/                   # Neo4j Browser screenshots used in the notebook
```

The raw `data/events.csv` (about 94 MB) is not uploaded. Download it from Kaggle and place it in `data/` to re-run the notebook.

## Periods

- **P1 (training):** 21 Aug - 29 Aug 2015, 157,544 events
- **P2 (testing):** 30 Aug - 07 Sep 2015, 155,128 events

Both end dates are inclusive. `timestamp` is Unix epoch in **milliseconds** (`pd.to_datetime(unit="ms")`, UTC).

## Pipeline

1. **Filter** events into P1 and P2 and save them to CSV (2.1 no. 1-2).
2. **Import P1 into Neo4j** in batches of 5,000: `Visitor` and `Item` nodes, with `VIEWED`, `ADDED_TO_CART` and `BOUGHT` relationships. Uniqueness constraints on `visitorId` and `itemId` (1.3).
3. **Subgraph composition** with Cypher: 146,603 nodes (91,525 Visitor, 55,078 Item) and 157,544 relationships, matching the CSV row count (2.1 no. 3).
4. **Vector Space Models** as sparse CSR matrices, weights `VIEWED=1`, `ADDED_TO_CART=3`, `BOUGHT=5` (2.1 no. 4-5):
   - User VSM: shape 91,525 x 55,078, 126,174 non-zero cells, density 0.0025%
   - Item VSM: shape 55,078 x 91,525, 126,174 non-zero cells, density 0.0025%
5. **Projection choice**: Item-Item (526,499 candidate edges vs 550,247 for User-User, fewer nodes, about 24 GB vs 67 GB if dense, denser vectors) (2.1 no. 6).
6. **Cosine similarity** for the reference item `187946` (most users); top-5 written back as `SIMILAR_TO {score}` and queried again, with before/after graph screenshots (2.1 no. 7-8).
7. **Item-based KNN (k = 5)** for a reproducible random user (`numpy.random.default_rng(42)`), then **Hit@5 / NDCG@5** against P2 (2.2).

## Results

- Selected user `142196`; 3 advertised items: `273541` (KNN), `187946` and `320130` (popularity fill). Hit@5 = 0 and NDCG@5 = 0 for this single user.
- Average over all 2,502 eligible users:
  - KNN only: Hit@5 = 0.0296, NDCG@5 = 0.0141
  - KNN + popularity fill: Hit@5 = 0.0304, NDCG@5 = 0.0143
  - Popularity only (baseline): Hit@5 = 0.0112, NDCG@5 = 0.0030
- Relevant items = items the user interacted with in P2 but not in P1.

## Run it

```powershell
uv venv myvenv --python 3.10
myvenv\Scripts\activate
uv pip install -r requirements.txt
```

Start a Neo4j instance (tested with Neo4j Desktop, `neo4j://127.0.0.1:7687`, database `neo4j`), open the notebook with the `myvenv` kernel and run the cells in order. The password is typed at a prompt or read from the `NEO4J_PASSWORD` environment variable; no credential is stored in the repo.

## Notes

- `requirements.txt` differs from the one in the exam sheet: the Neo4j driver must be **>= 5.14** because `execute_query` and `RoutingControl` do not exist in 4.2.1, and `nose` / `imdbpy` are not needed.
- The reset cell (`MATCH (n) DETACH DELETE n`) empties the database; do not re-run it after importing the data unless you want to re-import.
- Limitations: 83.4% of users have a single interaction in P1, so KNN often returns fewer than 5 items and a popularity fallback is used; each period is only 9 days; 96% of events are views, so relevance is a weak purchase signal.
