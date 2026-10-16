| Method | Test R@1 (%) | Test R@5 (%) | Test MRR@5 | Dev MRR@5 |
|---|---|---|---|---|
| BM25 (plain) | 69.2 | 82.1 | 0.738 | 0.698 |
| BM25 + normalization | 79.5 | 97.4 | 0.860 | 0.908 |
| Semantic (plain) | 66.7 | 84.6 | 0.741 | 0.746 |
| Semantic + normalization | 84.6 | 92.3 | 0.869 | 0.829 |
| Hybrid | 84.6 | 97.4 | 0.906 | 0.928 |

Test = organisers' 60 queries (39 answerable). Dev = 90 answerable train_dev queries. Embedding model: multilingual-e5-small.
