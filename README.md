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

3. Model registry: A centralized repository in MLOps (Machine Learning Operations) that manages the lifecycle of machine learning models, serving as a version control system to track and manage models from development to deployment.

4. Model launch tooling: Tools and processes designed to streamline the deployment of new machine learning models into production environments, often including steps for estimation, approval, preparation, scaling, and finalization, similar to how MLflow or Kubeflow help automate the transition of models from development to live service.

5. Model stability: Refers to a machine learning model's ability to produce consistent and accurate predictions even when faced with slight variations or noise in input data, ensuring its robustness and trustworthiness in real-world scenarios. For example, a stable facial recognition model will correctly identify a person even if their lighting or angle in the image changes slightly.

6. Model calibration: This process adjusts a model so that its predicted probabilities are reliable indicators of real-world likelihoods. For example, if a medical diagnosis model predicts a patient has a 70% chance of a certain condition, then among all patients for whom it predicts a 70% chance, approximately 70% should actually have that condition.

7. Model Normalized Entropy (NE): This metric evaluates how effectively our model reduces the uncertainty in its predictions compared to a very basic approach. It's calculated by taking our model's cross-entropy (which measures how close its predictions are to actual outcomes, with lower values indicating better performance) and dividing it by the cross-entropy of a simple baseline model that just predicts the average frequency of the event. A lower NE value indicates our model is significantly better at reducing prediction uncertainty.  

Real-world example: Imagine we have a model predicting whether an online ad will be clicked. If, on average, 5% of ads are clicked, a naive baseline model would always predict a 5% click probability for every ad. Our advanced model, however, makes specific predictions (e.g., 20% click chance for one ad, 1% for another). Normalized Entropy essentially tells us how much more precisely our model predicts individual ad clicks, significantly reducing the guesswork compared to just using the overall 5% average.

8. **Multi-task-multi-label (MTML) model**: This is a type of deep learning model, such as a **Transformer-based model** (common in NLP and computer vision), that's designed to perform several different classification tasks simultaneously. Within any of these tasks, a single item can also be associated with multiple categories or labels.

**Real-world Model and Example**:
Imagine a **Transformer-based MTML model** used by a social media platform to analyze user-uploaded images. This single model could:
1.  **Task 1: Object Recognition**: Identify multiple objects in the image (e.g., classifying a single image as containing `cat`, `tree`, and `sky`).
2.  **Task 2: Scene Classification**: Simultaneously categorize the image's environment (e.g., classifying it as `outdoor` and `daytime`).
3.  **Task 3: Safety Violation Detection**: While doing the above, also detect if the image contains `violence` and `graphic content` (multi-label within the safety task).

9. Mean Average Precision @ k (MAP@k)  
* **Core Idea:** Measures how well a system places *unique relevant items* at the *very top (ranks 1-k)* of its ranked lists, averaged across many searches.
* **Mental Model:** Users care most about the first few results. MAP@k rewards systems that consistently put the best answers right there.
* **Simple Example:**
    * Let's take `k = 3`
    * **Correct Answers:** `A, B`
    * **Our Top 3:** `A, C, A`
    * We get credit for the 1st `A`. `C` is wrong (no credit). The 2nd `A` is ignored because `A` was already credited. Only the *first unique correct* item counts.
