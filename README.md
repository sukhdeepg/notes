## Evaluation
1. Mean Average Precision @ k (MAP@k)  
* **Core Idea:** Measures how well a system places *unique relevant items* at the *very top (ranks 1-k)* of its ranked lists, averaged across many searches.
* **Mental Model:** Users care most about the first few results. MAP@k rewards systems that consistently put the best answers right there.
* **Simple Example:**
    * Let's take `k = 3`
    * **Correct Answers:** `A, B`
    * **Our Top 3:** `A, C, A`
    * We get credit for the 1st `A`. `C` is wrong (no credit). The 2nd `A` is ignored because `A` was already credited. Only the *first unique correct* item counts.

## Misc
1. `train_test_split`  
Separates our data into two sets for model training and evaluation:

* **`X_train`**: The features (input data) our model learns from.
* **`X_test`**: The features our model predicts on, but has never seen.
* **`y_train`**: The target labels/values corresponding to `X_train`, which our model learns to predict.
* **`y_test`**: The true target labels/values for `X_test`, used to evaluate our model's predictions.

This split prevents our model from "memorizing" the data and helps assess its performance on unseen information.

**Example:**

```python
from sklearn.model_selection import train_test_split
# X = features (e.g., house size), y = target (e.g., house price)
X = [[100], [150], [200], [250], [300]]
y = [100000, 150000, 200000, 250000, 300000]

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.4, random_state=42)

print(f"X_train (for training): {X_train}")
print(f"y_train (true values for training): {y_train}")
print(f"X_test (for testing): {X_test}")
print(f"y_test (true values for testing): {y_test}")
```

```
X_train (for training): [[200], [300], [100]]
y_train (true values for training): [200000, 300000, 100000]
X_test (for testing): [[250], [150]]
y_test (true values for testing): [250000, 150000]
```
2. Ranking funnel: In recommendation systems, this refers to a multi-stage process (sourcing, early-stage ranking, late-stage ranking) that progressively filters and orders items to be presented to a user, with each stage handling fewer candidates as operations become more computationally expensive.