### MI: Mechanistic Interpretability

1. Residual Stream
The main "highway" of information flow through a transformer model. Think of it as a river that carries information from input to output, with each layer adding or modifying information that flows through. Example: in GPT, the residual stream carries word representations that get progressively refined.

---

2. Superposition
When neural networks pack multiple features into fewer dimensions than would seem possible. Like storing 100 different concepts in a 50-dimensional space by using different sparse combinations. Example: a single neuron might activate for both "France" and "cooking" concepts simultaneously.

---

3. Polysemanticity
When individual neurons respond to multiple unrelated concepts. Example: one neuron might fire for both "the color blue" and "sad emotions" - it's doing double duty rather than having a single, interpretable function.