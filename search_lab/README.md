# Quick Reference Guide - Vector Search Evaluation

## 🚀 Quick Start (3 Steps)

### 1. Update Configuration
```python
CATALOG = "main"                          # YOUR CATALOG
SCHEMA = "default"                        # YOUR SCHEMA  
VECTOR_SEARCH_ENDPOINT = "vs_endpoint"    # YOUR ENDPOINT NAME
```

### 2. Run All Cells
- Ctrl+Shift+Enter (or Run All in Databricks)
- Estimated time: 15-30 minutes for 1K documents

### 3. View Results
- Check MLflow UI for metrics
- View visualizations in notebook
- Query results table: `{catalog}.{schema}.vector_search_eval_results`

---

## 📊 Key Metrics Explained

| Metric | Formula | Best Value | What It Means |
|--------|---------|------------|---------------|
| **Hit Rate@K** | # hits / # queries | 1.0 (100%) | Found correct doc in top-K |
| **MRR** | Avg(1/rank) | 1.0 | Quality of ranking |
| **Avg Rank** | Mean rank of hits | 1.0 | Typical position when found |

---

## 🔧 Quick Customizations

### Use Your Own Documents
```python
# Replace Wikipedia loading with:
documents = [
    {
        "id": "1",
        "title": "Your Document Title",
        "text": "Your document content...",
        "url": "https://..."
    },
    # ... more documents
]
```

### Change Evaluation Size
```python
SAMPLE_SIZE = 1000          # Number of docs to index
EVAL_SAMPLE_SIZE = 50       # Number of test queries
k_value = 5                 # Top-K retrieval
```

### Add More Models
```python
EMBEDDING_MODELS = [
    "databricks-gte-large-en",
    "databricks-bge-large-en",
    "sentence-transformers/all-MiniLM-L6-v2"  # Add more
]
```

---

## 🎯 Configuration Matrix

This notebook evaluates all combinations:

| Model | Retrieval Type | Total Configs |
|-------|----------------|---------------|
| databricks-gte-large-en | Dense | 1 |
| databricks-gte-large-en | Hybrid | 1 |
| databricks-bge-large-en | Dense | 1 |
| databricks-bge-large-en | Hybrid | 1 |
| **Total** | | **4** |

---

## 📈 Expected Runtime

| Component | Time (1K docs) | Notes |
|-----------|----------------|-------|
| Data Loading | 2-3 min | HuggingFace download |
| Index Creation | 1-2 min | Per index |
| Embedding Generation | 10-20 min | Faster with GPU |
| Evaluation | 2-5 min | Per configuration |
| **Total** | **~15-30 min** | Full pipeline |

---

## 🎨 Understanding Results

### Good Performance Indicators
- ✅ Hit Rate@5 > 70%
- ✅ MRR > 0.60
- ✅ Average Rank < 2.5

### Red Flags
- ⚠️ Hit Rate@5 < 50%
- ⚠️ MRR < 0.40
- ⚠️ Large variance across queries

### Common Patterns
- Hybrid typically beats Dense by 5-10%
- Both models usually perform similarly
- Performance varies by query complexity

---

## 🔍 Key Code Snippets

### Create Production Retriever
```python
from databricks.vector_search.client import VectorSearchClient

vsc = VectorSearchClient()
index = vsc.get_index("catalog.schema.index_name")

# Dense search
results = index.similarity_search(
    query_vector=embedding,
    columns=["id", "text", "title"],
    num_results=5
)

# Hybrid search  
results = index.similarity_search(
    query_vector=embedding,
    query_text=query,  # Add this for hybrid
    columns=["id", "text", "title"],
    num_results=5
)
```

### Query MLflow Results
```python
import mlflow

experiment = mlflow.get_experiment_by_name(EXPERIMENT_NAME)
runs = mlflow.search_runs(experiment_ids=[experiment.experiment_id])

# Get best run
best_run = runs.loc[runs['metrics.hit_rate@k'].idxmax()]
print(f"Best config: {best_run['params.embedding_model']}")
print(f"Hit rate: {best_run['metrics.hit_rate@k']}")
```

### Access Results Table
```python
results = spark.table("catalog.schema.vector_search_eval_results")

# Filter to specific config
gte_hybrid = results.filter(
    (col("model") == "databricks-gte-large-en") &
    (col("retrieval_type") == "hybrid")
)

# Calculate custom metrics
hit_rate = gte_hybrid.filter(col("hit@k") == True).count() / gte_hybrid.count()
```

---

## 🐛 Common Issues & Quick Fixes

### Issue: Endpoint Not Found
```python
# Create endpoint manually:
from databricks.vector_search.client import VectorSearchClient
vsc = VectorSearchClient()
vsc.create_endpoint(
    name="my_endpoint",
    endpoint_type="STANDARD"
)
```

### Issue: Out of Memory
```python
# Reduce batch size in embedding generation:
batch_size = 25  # Instead of 50

# Or reduce sample size:
SAMPLE_SIZE = 500  # Instead of 1000
```

### Issue: Slow Performance
```python
# Use GPU cluster:
# - g4dn.xlarge (AWS)
# - Standard_NC6s_v3 (Azure)
# - n1-standard-4 + 1 NVIDIA T4 (GCP)
```

### Issue: Index Creation Fails
```python
# Delete existing index first:
try:
    vsc.delete_index("catalog.schema.index_name")
except:
    pass

# Wait for deletion
import time
time.sleep(10)

# Then recreate
```

---

## 📊 MLflow UI Navigation

1. **View Experiment**: 
   - Navigate to MLflow in Databricks sidebar
   - Find experiment: `{your_username}/vector_search_evaluation`

2. **Compare Runs**:
   - Select multiple runs
   - Click "Compare"
   - View metric charts

3. **View Artifacts**:
   - Click on a run
   - Scroll to "Artifacts" section
   - Download visualizations

---

## 🎯 Decision Guide

### Choose Dense Retrieval When:
- Queries are conceptual/semantic
- Storage/compute costs are critical
- Exact term matching is less important
- Query latency must be minimal

### Choose Hybrid Search When:
- Queries contain specific terms/names
- Accuracy is more important than speed
- Budget allows for extra compute
- Users expect exact keyword matches

### Choose GTE vs BGE:
- **GTE**: Slightly better for general domains
- **BGE**: Slightly better for technical content
- **Reality**: Performance is usually similar (~1-3% difference)

---

## 💾 Cleanup After Testing

```python
from databricks.vector_search.client import VectorSearchClient

vsc = VectorSearchClient()

# Delete indexes
for model in EMBEDDING_MODELS:
    index_name = f"{CATALOG}.{SCHEMA}.{TABLE_NAME}_{model.replace('-', '_')}_index"
    try:
        vsc.delete_index(index_name)
        print(f"Deleted {index_name}")
    except:
        pass

# Delete endpoint (if you created it)
# vsc.delete_endpoint(VECTOR_SEARCH_ENDPOINT)

# Drop tables
spark.sql(f"DROP TABLE IF EXISTS {CATALOG}.{SCHEMA}.{TABLE_NAME}")
spark.sql(f"DROP TABLE IF EXISTS {CATALOG}.{SCHEMA}.vector_search_eval_results")
```

---

## 📚 Useful Links

- [Vector Search Docs](https://docs.databricks.com/en/generative-ai/vector-search.html)
- [MLflow Guide](https://mlflow.org/docs/latest/llms/index.html)
- [Evaluation Metrics](https://docs.databricks.com/en/generative-ai/agent-evaluation/index.html)

---

## 🚀 Next Steps After Demo

1. **Scale**: Test with 10K+ documents
2. **Real Queries**: Use actual user queries
3. **Add Reranking**: Cross-encoder for top results
4. **Deploy**: Integrate best config into production
5. **Monitor**: Track real-world performance

---

**Pro Tip**: Start with default settings, analyze results, then customize based on your specific needs!
