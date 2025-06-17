1. Fast Evals
These are evaluation methods designed to quickly assess AI model performance without requiring expensive, time-consuming processes. For example, instead of running a full human evaluation study that takes weeks, we might use automated metrics or smaller representative test sets that give reliable results in minutes or hours.

---

2. PMF
Stands for "Probability Mass Function" - a statistical concept used in evaluations to describe the probability distribution of discrete outcomes. In AI evals, we might use PMF to analyze how likely a model is to produce different types of responses.

---

3. LLM Judge Issues:
- **Inefficient algorithm choice**: Using computationally expensive methods when simpler ones would work. Example: using a massive 70B parameter model as a judge when a 7B model gives equivalent evaluation quality.
- **Late binding closure**: A programming concept where variables are resolved at runtime rather than compile time, potentially causing unexpected behavior in evaluation pipelines.

---

4. Flesch-Kincaid
A readability scoring system that measures text complexity based on sentence length and syllable count. In AI evals, it's used to assess whether model outputs are appropriately complex for intended audiences.