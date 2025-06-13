1. `train_test_split`: Separates our data into two sets for model training and evaluation:

    * **`X_train`**: The features (input data) our model learns from.
    * **`X_test`**: The features our model predicts on, but has never seen.
    * **`y_train`**: The target labels/values corresponding to `X_train`, which our model learns to predict.
    * **`y_test`**: The true target labels/values for `X_test`, used to evaluate our model's predictions.

    * This split prevents our model from "memorizing" the data and helps assess its performance on unseen information.

    * **Example:**

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

---

2. Ranking funnel: In recommendation systems, this refers to a multi-stage process (sourcing, early-stage ranking, late-stage ranking) that progressively filters and orders items to be presented to a user, with each stage handling fewer candidates as operations become more computationally expensive.

---

3. Model registry: A centralized repository in MLOps (Machine Learning Operations) that manages the lifecycle of machine learning models, serving as a version control system to track and manage models from development to deployment.

---

4. Model launch tooling: Tools and processes designed to streamline the deployment of new machine learning models into production environments, often including steps for estimation, approval, preparation, scaling, and finalization, similar to how MLflow or Kubeflow help automate the transition of models from development to live service.

---

5. Model stability: Refers to a machine learning model's ability to produce consistent and accurate predictions even when faced with slight variations or noise in input data, ensuring its robustness and trustworthiness in real-world scenarios. For example, a stable facial recognition model will correctly identify a person even if their lighting or angle in the image changes slightly.

---

6. Model calibration: This process adjusts a model so that its predicted probabilities are reliable indicators of real-world likelihoods. For example, if a medical diagnosis model predicts a patient has a 70% chance of a certain condition, then among all patients for whom it predicts a 70% chance, approximately 70% should actually have that condition.

---

7. Model Normalized Entropy (NE)
    Measures how "confused" or uncertain a model's predictions are, scaled relative to the maximum possible confusion.

    **Intuition:** 
    - Regular entropy tells us how uncertain predictions are
    - But a model predicting between 2 classes vs 1000 classes will have very different entropy ranges
    - Normalized entropy divides by the maximum possible entropy, giving us a 0-1 scale where:
    - 0 = perfectly confident (no uncertainty)
    - 1 = maximally confused (uniform random guessing)

    The key insight: normalized entropy gives us a universal "confusion score" that's comparable across different models and tasks, regardless of how many classes they're predicting.

    ```python
    import torch
    import torch.nn.functional as F

    def normalized_entropy(logits):
        # LOGITS: Raw scores from neural network (can be any numbers, positive/negative)
        # Think of them as "raw confidence scores" before converting to probabilities
        
        # SOFTMAX: Converts logits into probabilities that sum to 1
        # Higher logits become higher probabilities
        # dim=-1 means "apply softmax along the last dimension" (across classes)
        probs = F.softmax(logits, dim=-1)
        
        # ENTROPY: Measures uncertainty/randomness in the probabilities
        # Formula: -sum(p * log(p)) where p is each probability
        # 1e-8 prevents log(0) which would be -infinity
        entropy = -torch.sum(probs * torch.log(probs + 1e-8), dim=-1)
        
        # MAX ENTROPY: Highest possible entropy (when all probabilities are equal)
        # For N classes, max entropy = log(N)
        num_classes = logits.size(-1)  # size(-1) gets the last dimension size
        max_entropy = torch.log(torch.tensor(num_classes, dtype=torch.float))
        
        # NORMALIZE: Scale entropy to 0-1 range by dividing by maximum
        normalized = entropy / max_entropy
        return normalized

    # [[...]] creates a 2D tensor: 1 sample with 3 class scores

    # Confident prediction (low entropy)
    confident_logits = torch.tensor([[10.0, 1.0, 1.0]])  # Class 0 has much higher score
    print(f"Confident prediction: {normalized_entropy(confident_logits):.3f}")

    # Uncertain prediction (high entropy) 
    uncertain_logits = torch.tensor([[1.0, 1.0, 1.0]])   # All classes have same score
    print(f"Uncertain prediction: {normalized_entropy(uncertain_logits):.3f}")

    # What's happening internally:
    print("\nStep-by-step for confident prediction:")
    logits = torch.tensor([[10.0, 1.0, 1.0]])
    probs = F.softmax(logits, dim=-1)
    print(f"Logits: {logits}")
    print(f"Probabilities after softmax: {probs}")
    print(f"These probabilities sum to: {probs.sum():.3f}")

    # Output:
    # Confident prediction: 0.368
    # Uncertain prediction: 1.000
    # 
    # Step-by-step for confident prediction:
    # Logits: tensor([[10., 1., 1.]])
    # Probabilities after softmax: tensor([[0.9999, 0.0000, 0.0000]])
    # These probabilities sum to: 1.000
    ```

    **Logits represent "raw confidence scores" before converting to probabilities:**

    - **[10.0, 1.0, 1.0]** = "I'm VERY confident it's class 0"
    - Class 0 gets score 10, others get score 1
    - After softmax: ~[99.99%, 0.00%, 0.00%] 
    - The model is almost certain → low entropy

    - **[1.0, 1.0, 1.0]** = "I have no idea, all classes seem equally likely"
    - All classes get the same score
    - After softmax: [33.33%, 33.33%, 33.33%]
    - Maximum uncertainty → high entropy (close to 1.0)

    **The key insight:** It's not the absolute values that matter, it's the *differences* between them:

    - **Large differences** in logits → confident predictions → low entropy
    - **Small/no differences** in logits → uncertain predictions → high entropy

    We could also have:
    - **[100, 1, 1]** → even more confident (entropy closer to 0)
    - **[2, 1, 1]** → somewhat confident (entropy between 0 and 1)
    - **[0, 0, 0]** → completely uncertain (entropy = 1.0, same as [1,1,1])

    The softmax function automatically converts these raw scores into proper probabilities that sum to 1, but it preserves the relative confidence relationships.

---

8. **Multi-task-multi-label (MTML) model**: This is a type of deep learning model, such as a **Transformer-based model** (common in NLP and computer vision), that's designed to perform several different classification tasks simultaneously. Within any of these tasks, a single item can also be associated with multiple categories or labels.

* **Real-world Model and Example**:
Imagine a **Transformer-based MTML model** used by a social media platform to analyze user-uploaded images. This single model could:
    *  **Task 1: Object Recognition**: Identify multiple objects in the image (e.g., classifying a single image as containing `cat`, `tree`, and `sky`).
    *  **Task 2: Scene Classification**: Simultaneously categorize the image's environment (e.g., classifying it as `outdoor` and `daytime`).
    *  **Task 3: Safety Violation Detection**: While doing the above, also detect if the image contains `violence` and `graphic content` (multi-label within the safety task).

---

9. Mean Average Precision @ k (MAP@k)  
* **Core Idea:** Measures how well a system places *unique relevant items* at the *very top (ranks 1-k)* of its ranked lists, averaged across many searches.
* **Mental Model:** Users care most about the first few results. MAP@k rewards systems that consistently put the best answers right there.
* **Simple Example:**
    * Let's take `k = 3`
    * **Correct Answers:** `A, B`
    * **Our Top 3:** `A, C, A`
    * We get credit for the 1st `A`. `C` is wrong (no credit). The 2nd `A` is ignored because `A` was already credited. Only the *first unique correct* item counts.

---

10. Data Categories
* **Nominal** 
    * **Definition**: Categories with no inherent order
    * **Examples**: Colors (red, blue, green), Gender (male, female), Car brands (Toyota, BMW, Ford)
    * **Key point**: We can't rank or order these meaningfully

* **Ordinal**
    * **Definition**: Categories with a clear order/ranking
    * **Examples**: Education level (High School < Bachelor's < Master's < PhD), Movie ratings (1-5 stars), Survey responses (Poor < Fair < Good < Excellent)
    * **Key point**: Order matters, but gaps between levels aren't necessarily equal

* **Cardinal** (Numerical)
    * **Definition**: Numbers where we can perform mathematical operations
    * **Examples**: Age (25, 30, 45), Income ($50,000, $75,000), Temperature (20°C, 25°C)
    * **Key point**: We can add, subtract, calculate averages

* **Related Terms:**
    * **Interval**: Numbers with equal gaps but no true zero (Temperature in Celsius - 0°C doesn't mean "no temperature")
    * **Ratio**: Numbers with equal gaps AND true zero (Height, Weight, Income - 0 means absence)

* **Machine Learning Impact:**
    * **Nominal → One-hot encoding**
        * Problem: ML algorithms can't understand "red" vs "blue" 
        * Solution: Create separate binary columns (is_red: 0/1, is_blue: 0/1)
        * Example: Color [red, blue] → red_0, red_1, blue_0, blue_1

    * **Nominal (High Cardinality) → Binary encoding**
        * Problem: One-hot creates too many columns (100 categories = 100 columns)
        * Solution: Convert to binary representation using fewer columns
        * Example: 8 categories need only 3 binary columns (2³=8): Cat1→001, Cat2→010, Cat3→011
        * Example: Cities [NYC, LA, Chicago, Miami] → 2 binary columns can represent 4 cities:
            * NYC: [0,0], LA: [0,1], Chicago: [1,0], Miami: [1,1]

   * **Nominal (Target-Correlated) → Target encoding**
        * Problem: Some categories are much better predictors than others, but one-hot encoding treats them all equally
        * Example: We're predicting whether customers will buy a product (target: 0=No, 1=Yes)
        * Step 1 - Our training data looks like this:
            ```
            Customer_Job    | Will_Buy (Target)
            Doctor         | 1
            Doctor         | 1  
            Doctor         | 0
            Teacher        | 0
            Teacher        | 0
            Teacher        | 1
            Unemployed     | 0
            Unemployed     | 0
            Unemployed     | 0
            ```
        * Step 2 - Calculate average target for each category:
            ```python
            # What we calculate behind the scenes:
            Doctor: (1+1+0)/3 = 0.67     # 67% of doctors buy
            Teacher: (0+0+1)/3 = 0.33     # 33% of teachers buy  
            Unemployed: (0+0+0)/3 = 0.00  # 0% of unemployed buy
            ```
        * Step 3 - Replace categories with these calculated averages:
            ```
            Original: [Doctor, Teacher, Unemployed, Doctor]
            Encoded:  [0.67,  0.33,   0.00,      0.67]
            ```
        * Why this works: The model now sees that Doctor=0.67 (high chance of buying) vs Unemployed=0.00 (low chance)
        * Risk: If we only had 1 doctor in training who didn't buy, we'd wrongly encode all doctors as 0.00
        * Code example:
            ```python
            # Calculate target encoding
            target_means = df.groupby('Job')['Will_Buy'].mean()
            df['Job_encoded'] = df['Job'].map(target_means)
            ```

    * **Ordinal → Label encoding** 
        * Problem: Need to preserve the ranking order
        * Solution: Convert to numbers that maintain order (Poor=1, Fair=2, Good=3)
        * Example: Education [High School, Bachelor's, Master's] → [1, 2, 3]

    * **Cardinal → Use directly or scale**
        * Problem: Different scales can dominate (Age: 25 vs Income: 50000)
        * Solution: Use raw values or normalize to similar ranges
        * Example: Age [25, 30] and Income [50k, 60k] → normalize both to 0-1 scale

    * **Why this matters**: Wrong encoding breaks algorithms - treating nominal as numbers (red=1, blue=2) makes the model think blue > red, which is meaningless!
