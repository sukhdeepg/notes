## Evaluation
1. Mean Average Precision @ k (MAP@k)  
* **Core Idea:** Measures how well a system places *unique relevant items* at the *very top (ranks 1-k)* of its ranked lists, averaged across many searches.
* **Mental Model:** Users care most about the first few results. MAP@k rewards systems that consistently put the best answers right there.
* **Simple Example:**
    * Let's take `k = 3`
    * **Correct Answers:** `A, B`
    * **Our Top 3:** `A, C, A`
    * We get credit for the 1st `A`. `C` is wrong (no credit). The 2nd `A` is ignored because `A` was already credited. Only the *first unique correct* item counts.
