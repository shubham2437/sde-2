# Activation Functions — Full Study Guide & Deep Dive

*Deep Learning Roadmap · Phase 1: Neural Network Core*

> Every function is drawn as a **plain-text plot** right below it — `*` traces the function f(x) and `.` traces its derivative f '(x). No images; renders anywhere.

---

## 1. Why do we even need activation functions?

An activation function is the small **non-linear** step applied to a neuron's weighted sum. It is the single most important reason neural networks can learn anything more interesting than a straight line.

### The core reason: without them, depth is pointless

A neuron computes `z = w·x + b` — a **linear** operation. If you stack many layers but apply no non-linearity between them, the whole network collapses into one linear function:

```
layer2(layer1(x)) = W2(W1x + b1) + b2 = (W2W1)x + (W2b1 + b2) = W'x + b'
```

That is *still just a single linear layer*. No matter how many layers you add, a purely linear network can only draw a straight-line (hyperplane) boundary — exactly the limitation of a single perceptron. **The activation function injects non-linearity**, so each layer can bend and reshape the data, and stacking layers genuinely increases the network's power.

> **Universal Approximation Theorem.** A network with just one hidden layer and a non-linear activation can approximate *any* continuous function to arbitrary accuracy, given enough neurons. Remove the activation and the theorem collapses.

### What else activation functions do

- **Introduce non-linearity** — the essential job, letting networks model curved, complex relationships.
- **Control the output range** — e.g. squash to (0,1) for probabilities, or (−1,1) to keep signals centered.
- **Decide which neurons "fire"** — sparse activations (ReLU zeroing negatives) make networks efficient.
- **Shape the gradients** — training uses backpropagation, so the activation's *derivative* controls how gradient signal flows backward. A bad choice causes **vanishing** or **exploding** gradients.

---

## 2. What makes a *good* activation function?

| Property | Why it matters |
|----------|----------------|
| **Non-linear** | The whole point — enables depth and complex decision boundaries. |
| **Differentiable** | Backpropagation needs a gradient; a useful, non-zero derivative lets the network learn. |
| **Non-saturating gradient** | If the derivative goes to ~0 over large input ranges, gradients vanish and learning stalls. |
| **Zero-centered output** | Outputs around 0 keep gradient updates balanced; all-positive outputs cause zig-zagging. |
| **Computationally cheap** | Applied millions of times per pass — simple functions (ReLU) train much faster. |
| **Monotonic / smooth** | Often helps optimization behave predictably. |

The two big families: **bounded/saturating** functions (step, sigmoid, tanh) squash inputs into a fixed range; the **ReLU family and modern smooth variants** are unbounded above and dominate today's deep networks.

---

## 3. Every activation function — deep dive

### Binary Step  *(legacy)*

```text
  1.5 |                          |                          
      |                          |                          
      |                          |                          
      |                          |                          
      |                          |**************************
      |                          |                          
      |                          |                          
      |                          |                          
      |                          |                          
      |                          |                          
      |                          |                          
 -0.0 |***************************--------------------------
      |                          |                          
      |                          |                          
 -0.4 |                          |                          
       +-----------------------------------------------------
       -6                                                 6   (x)
  (* = f(x))
```

**Formula:** `f(x) = 1 if x > 0, else 0`
**Range:** {0, 1} · **Derivative:** 0 everywhere (undefined at 0)
**Purpose:** the original perceptron activation — a hard yes/no threshold. Historically important, but its derivative is zero everywhere, so it is useless for gradient-based training.

- **Pros:** Simple, intuitive binary output; very cheap.
- **Cons:** No usable gradient → can't learn via backprop; can't do multi-class or probabilities.

### Linear / Identity  *(output only)*

```text
  3.5 |                          |                          
      |                          |                       ***
      |                          |                   ****   
      |                          |               ****       
      |                          |          *****           
      |                          |      ****                
      |.............................****....................
  0.0 |------------------------*****------------------------
      |                    ****  |                          
      |                ****      |                          
      |           *****          |                          
      |       ****               |                          
      |   ****                   |                          
      |***                       |                          
 -3.5 |                          |                          
       +-----------------------------------------------------
       -6                                                 6   (x)
  (* = f(x),  . = f '(x))
```

**Formula:** `f(x) = a·x`
**Range:** (−∞, ∞) · **Derivative:** constant (a)
**Purpose:** used in the **output layer of regression** models, where you want an unbounded real number. Never use it in hidden layers — a network of linear activations collapses to a single linear layer.

- **Pros:** Correct for regression outputs; preserves scale; trivially cheap.
- **Cons:** No non-linearity; constant gradient carries no information about x; useless in hidden layers.

### Sigmoid (Logistic)  *(binary output; legacy in hidden layers)*

```text
  1.3 |                          |                          
      |                          |                          
      |                          |                  ********
      |                          |        **********        
      |                          |     ***                  
      |                          |  ***                     
      |                          |**                        
      |                          *                          
      |                        **|                          
      |                     ***.....                        
      |                  ***...  |  .....                   
      |        **********.       |       ..........         
 -0.0 |********.-----------------+-----------------.........
      |                          |                          
 -0.3 |                          |                          
       +-----------------------------------------------------
       -6                                                 6   (x)
  (* = f(x),  . = f '(x))
```

**Formula:** `f(x) = 1 / (1 + e^-x)`
**Range:** (0, 1) · **Derivative:** `f(x)·(1 − f(x))`, max value only **0.25**
**Purpose:** squashes any input into (0, 1), so it reads as a **probability**. Still the standard choice for the **output of binary classification** and for gates in LSTMs. Once popular in hidden layers, now largely replaced there.

- **Pros:** Smooth, bounded, probabilistic interpretation; great for binary output.
- **Cons:** Saturates at both ends → **vanishing gradients**; not zero-centered (slows learning); `e^-x` is costly.

### Tanh (Hyperbolic Tangent)  *(hidden / RNN)*

```text
  1.4 |                          |                          
      |                          |                          
      |                         ...      *******************
      |                        . | . ****                   
      |                       .  |  *                       
      |                     ..   | * ..                     
      |                   ..     |*    ..                   
  0.0 |...................-------*-------...................
      |                         *|                          
      |                        * |                          
      |                       *  |                          
      |                   ****   |                          
      |*******************       |                          
      |                          |                          
 -1.4 |                          |                          
       +-----------------------------------------------------
       -6                                                 6   (x)
  (* = f(x),  . = f '(x))
```

**Formula:** `f(x) = (e^x − e^-x) / (e^x + e^-x)`
**Range:** (−1, 1) · **Derivative:** `1 − f(x)^2`, max value **1.0**
**Purpose:** a **zero-centered** version of sigmoid. Because outputs center around 0, gradient updates are better behaved, so tanh beats sigmoid for hidden layers when a bounded activation is wanted (and inside RNNs/LSTMs).

- **Pros:** Zero-centered; stronger gradients than sigmoid; smooth and bounded.
- **Cons:** Still saturates at the tails → vanishing gradients in deep nets; still uses exponentials.

### ReLU (Rectified Linear Unit)  *(hidden layer default)*

```text
  6.6 |                          |                          
      |                          |                        **
      |                          |                      **  
      |                          |                   ***    
      |                          |                 **       
      |                          |              ***         
      |                          |            **            
      |                          |         ***              
      |                          |       **                 
      |                          |    ***                   
      |                          |..**......................
  0.2 |*****************************------------------------
      |                          |                          
      |                          |                          
 -1.6 |                          |                          
       +-----------------------------------------------------
       -6                                                 6   (x)
  (* = f(x),  . = f '(x))
```

**Formula:** `f(x) = max(0, x)`
**Range:** [0, ∞) · **Derivative:** 1 if x>0, else 0
**Purpose:** the **default hidden-layer activation** for most modern networks (CNNs, MLPs). It doesn't saturate for positive inputs, so gradients flow freely and training is fast; its sparsity (zeroing negatives) is efficient.

- **Pros:** Cheap and fast; no positive-side saturation → strong gradient flow; sparse activations.
- **Cons:** **"Dying ReLU"** — neurons stuck on negative inputs output 0 forever (zero gradient); not zero-centered; unbounded output can need care.

### Leaky ReLU & PReLU  *(hidden)*

```text
  6.6 |                          |                          
      |                          |                        **
      |                          |                     ***  
      |                          |                   **     
      |                          |                ***       
      |                          |             ***          
      |                          |           **             
      |                          |        ***               
      |                          |     ***                  
      |                          |...**.....................
      |...........................***                       
 -0.2 |---************************--------------------------
      |***                       |                          
      |                          |                          
 -2.1 |                          |                          
       +-----------------------------------------------------
       -6                                                 6   (x)
  (* = f(x),  . = f '(x))
```

**Formula:** `f(x) = x if x > 0, else α·x`  (α ≈ 0.01–0.1)
**Range:** (−∞, ∞) · **Derivative:** 1 if x>0, else α
**Purpose:** fixes **dying ReLU** by giving negatives a small non-zero slope, so those neurons keep a gradient and can recover. **PReLU** is the same idea but *learns* α during training instead of fixing it.

- **Pros:** No dead neurons; keeps ReLU's speed and simplicity; PReLU adapts α to the data.
- **Cons:** α is an extra hyperparameter; improvement over plain ReLU is often modest; PReLU adds parameters that can overfit.

### ELU (Exponential Linear Unit) & SELU  *(hidden)*

```text
  6.6 |                          |                          
      |                          |                        **
      |                          |                     ***  
      |                          |                   **     
      |                          |                ***       
      |                          |             ***          
      |                          |          ***             
      |                          |        **                
      |                          |     ***                  
      |                        .....***.....................
  0.3 |........................--***------------------------
      |                      ****|                          
      |**********************    |                          
      |                          |                          
 -2.2 |                          |                          
       +-----------------------------------------------------
       -6                                                 6   (x)
  (* = f(x),  . = f '(x))
```

**Formula:** `f(x) = x if x > 0, else α·(e^x − 1)`
**Range:** (−α, ∞) · **Derivative:** 1 if x>0, else `f(x)+α`
**Purpose:** a smooth curve for negatives that pushes the mean activation toward zero (more zero-centered than ReLU), which can speed learning. **SELU** is a scaled variant that *self-normalizes* activations, keeping them stable across layers.

- **Pros:** Smooth; closer to zero-centered than ReLU; no hard dead zone; SELU self-normalizes.
- **Cons:** The exponential makes it slower than ReLU; SELU needs specific weight init/architecture to work as intended.

### GELU (Gaussian Error Linear Unit)  *(transformers)*

```text
  6.6 |                          |                          
      |                          |                        **
      |                          |                      **  
      |                          |                   ***    
      |                          |                 **       
      |                          |              ***         
      |                          |            **            
      |                          |         ***              
      |                          |       **                 
      |                          |     **                   
      |                          |  ***                     
  0.2 |********************-----****------------------------
      |                    ***** |                          
      |                          |                          
 -1.6 |                          |                          
       +-----------------------------------------------------
       -6                                                 6   (x)
  (* = f(x))
```

**Formula:** `f(x) = x · Φ(x)`  (Φ = standard normal CDF)
**Range:** ≈ (−0.17, ∞) · **Derivative:** smooth, non-monotonic near 0
**Purpose:** the **standard activation in Transformers** (BERT, GPT). Instead of a hard 0/1 gate, it weights each input by the probability it's positive — a smooth, probabilistic version of ReLU. Excellent for large language and vision models.

- **Pros:** Smooth; strong empirical performance in deep and attention-based models; the de-facto transformer choice.
- **Cons:** More expensive than ReLU; the exact form uses an error-function/approximation.

### Swish / SiLU  *(modern)*

```text
  6.6 |                          |                          
      |                          |                        **
      |                          |                      **  
      |                          |                   ***    
      |                          |                 **       
      |                          |               **         
      |                          |            ***           
      |                          |          **              
      |                          |        **                
      |                          |     ***                  
      |                          |   **                     
  0.2 |************-------------*****-----------------------
      |            ************* |                          
      |                          |                          
 -1.6 |                          |                          
       +-----------------------------------------------------
       -6                                                 6   (x)
  (* = f(x))
```

**Formula:** `f(x) = x · sigmoid(x)`
**Range:** ≈ (−0.28, ∞) · **Derivative:** smooth, non-monotonic
**Purpose:** discovered by automated search at Google; a smooth, self-gated function that often **edges out ReLU** on deep networks (used in EfficientNet). The small dip below zero and smoothness help gradient flow.

- **Pros:** Smooth; unbounded above; frequently beats ReLU on deep nets; self-gating.
- **Cons:** Costlier than ReLU (uses a sigmoid); the improvement is task-dependent, not guaranteed.

### Softplus  *(niche)*

```text
  6.5 |                          |                          
      |                          |                        **
      |                          |                      **  
      |                          |                    **    
      |                          |                 ***      
      |                          |               **         
      |                          |            ***           
      |                          |          **              
      |                          |       ***                
      |                          |    ***                   
      |                          | ***    ..................
      |                      ******.......                  
  0.1 |**********************.---+--------------------------
      |                          |                          
 -1.0 |                          |                          
       +-----------------------------------------------------
       -6                                                 6   (x)
  (* = f(x),  . = f '(x))
```

**Formula:** `f(x) = ln(1 + e^x)`
**Range:** (0, ∞) · **Derivative:** `sigmoid(x)`
**Purpose:** a smooth approximation of ReLU (its derivative is exactly the sigmoid). Useful where you need a strictly positive, everywhere-differentiable output — e.g. predicting a variance or a rate parameter that must stay positive.

- **Pros:** Smooth everywhere; always positive; nice when a positive, differentiable output is required.
- **Cons:** Slower than ReLU; rarely better in practice for hidden layers; can lose ReLU's exact sparsity.

### Mish  *(modern)*

```text
  6.6 |                          |                          
      |                          |                        **
      |                          |                      **  
      |                          |                   ***    
      |                          |                 **       
      |                          |              ***         
      |                          |            **            
      |                          |         ***              
      |                          |       **                 
      |                          |     **                   
      |                          |  ***                     
  0.2 |************--------------***------------------------
      |            **************|                          
      |                          |                          
 -1.6 |                          |                          
       +-----------------------------------------------------
       -6                                                 6   (x)
  (* = f(x))
```

**Formula:** `f(x) = x · tanh(softplus(x)) = x · tanh(ln(1 + e^x))`
**Range:** ≈ (−0.31, ∞) · **Derivative:** smooth, non-monotonic
**Purpose:** a smooth, self-regularizing activation similar in spirit to Swish; has shown accuracy gains in some computer-vision models (e.g. YOLOv4). Smoothness and the small negative region help preserve information.

- **Pros:** Smooth; strong results on some vision tasks; preserves small negative information.
- **Cons:** The most expensive of the common options; gains are not universal.

### Softmax  *(multi-class output)*

**Formula:** `f(xi) = e^xi / Σj e^xj`
**Range:** (0, 1), and all outputs **sum to 1**  *(operates over a whole vector, so there is no single-input curve to plot)*
**Purpose:** the standard **output activation for multi-class classification**. It turns a vector of raw scores (logits) into a probability distribution over classes — the largest score gets the largest probability. Pairs with cross-entropy loss.
**Example:** logits `[2.0, 1.0, 0.1]` → softmax ≈ `[0.66, 0.24, 0.10]`.

- **Pros:** Gives interpretable, normalized class probabilities; pairs cleanly with cross-entropy loss.
- **Cons:** Output layer only (operates over a whole vector, not one value); sensitive to very large logits (needs numerical stabilization).

---

## 4. The vanishing-gradient problem (why ReLU won)

Backpropagation multiplies derivatives layer by layer. If an activation's derivative is small (much less than 1), those small numbers **multiply together** across many layers and the gradient shrinks toward zero — the early layers barely learn. This is the **vanishing gradient problem**.

Sigmoid's derivative peaks at just **0.25**, and tanh's at 1.0 but only at x=0 — both flatten to ~0 in their tails. Stack 10 such layers and the signal all but disappears. **ReLU's derivative is exactly 1 for all positive inputs**, so gradients pass through undiminished. This single property is the main reason ReLU and its relatives replaced sigmoid/tanh in deep hidden layers.

```text
 gradient of the two derivatives (compare the heights):

 1.0  ReLU  f'(x) = 1  ******************************   (stays at 1 for x>0)

0.25  Sigmoid f'(x) peaks here  ...**...                (and shrinks to ~0 at the tails)
 0.0  .....**............................**.....
                          x=0
```

---

## 5. Practical guide — which one to use

| Situation | Recommended | Why |
|-----------|-------------|-----|
| Hidden layers (default start) | **ReLU** | Fast, reliable, no positive saturation. The safe default. |
| Hidden layers, ReLU dying | Leaky ReLU / ELU | Keeps a gradient on the negative side so neurons recover. |
| Transformers / attention models | **GELU** (or SiLU) | Proven choice for BERT, GPT and similar. |
| Extra accuracy on deep CNNs | Swish / Mish | Smooth variants that sometimes beat ReLU. |
| RNN / LSTM internals | Tanh + Sigmoid | Bounded, zero-centered states and gates. |
| **Output:** binary classification | **Sigmoid** | One probability in (0, 1). |
| **Output:** multi-class classification | **Softmax** | Probability distribution summing to 1. |
| **Output:** regression | **Linear** | Unbounded real-valued prediction. |

> **Rule of thumb:** start hidden layers with **ReLU**; switch to **Leaky ReLU / ELU** if neurons die; use **GELU** for transformers. Pick the **output** activation from the *task*: sigmoid (binary), softmax (multi-class), linear (regression).

---

## 6. Cheat-sheet summary

| Function | Formula | Range | Main use | Key weakness |
|----------|---------|-------|----------|--------------|
| Step | 1 if x>0 else 0 | {0,1} | Perceptron (historic) | No gradient |
| Linear | a·x | (−∞,∞) | Regression output | No non-linearity |
| Sigmoid | 1/(1+e^-x) | (0,1) | Binary output | Vanishing gradient |
| Tanh | (e^x−e^-x)/(e^x+e^-x) | (−1,1) | RNN hidden states | Still saturates |
| ReLU | max(0,x) | [0,∞) | Default hidden layer | Dying neurons |
| Leaky ReLU | x or α·x | (−∞,∞) | Fixes dying ReLU | Extra hyperparam |
| ELU | x or α(e^x−1) | (−α,∞) | Zero-centered hidden | Slower (exp) |
| GELU | x·Φ(x) | ≈(−0.17,∞) | Transformers | Costlier |
| Swish/SiLU | x·sigmoid(x) | ≈(−0.28,∞) | Deep CNNs | Costlier |
| Softplus | ln(1+e^x) | (0,∞) | Positive outputs | Slow, rarely best |
| Mish | x·tanh(ln(1+e^x)) | ≈(−0.31,∞) | Some vision nets | Most expensive |
| Softmax | e^xi/Σe^xj | (0,1), Σ=1 | Multi-class output | Output layer only |

> **One-line summary:** Activation functions add the non-linearity that makes deep networks powerful. Their *derivatives* decide whether training works. Use **ReLU-family** functions in hidden layers to keep gradients flowing, and pick the **output** activation to match your task.

---

## 7. Pros & Cons — at a glance

| Function | Pros | Cons |
|----------|------|------|
| **Step** | Simple, intuitive binary output; very cheap | No usable gradient → can't train with backprop; no multi-class/probabilities |
| **Linear** | Correct for regression outputs; preserves scale; trivially cheap | No non-linearity; constant gradient carries no info about x; useless in hidden layers |
| **Sigmoid** | Smooth, bounded; probabilistic (0,1); great for binary output | Saturates → vanishing gradients; not zero-centered; `e^-x` is costly |
| **Tanh** | Zero-centered; stronger gradients than sigmoid; smooth & bounded | Still saturates at tails → vanishing gradients; uses exponentials |
| **ReLU** | Cheap & fast; no positive saturation → strong gradient flow; sparse | Dying ReLU (dead neurons); not zero-centered; unbounded output |
| **Leaky ReLU / PReLU** | No dead neurons; keeps ReLU's speed; PReLU adapts α | Extra hyperparameter α; gains over ReLU often modest; PReLU can overfit |
| **ELU / SELU** | Smooth; closer to zero-centered; no hard dead zone; SELU self-normalizes | Exponential makes it slower; SELU needs specific init/architecture |
| **GELU** | Smooth; strong in deep/attention models; transformer default | More expensive than ReLU; uses error-function/approximation |
| **Swish / SiLU** | Smooth; unbounded above; often beats ReLU on deep nets; self-gating | Costlier than ReLU; improvement is task-dependent |
| **Softplus** | Smooth everywhere; always positive; differentiable | Slower than ReLU; rarely best in hidden layers; loses exact sparsity |
| **Mish** | Smooth; strong on some vision tasks; preserves small negatives | Most expensive common option; gains not universal |
| **Softmax** | Interpretable, normalized class probabilities; pairs with cross-entropy | Output layer only (works over a vector); sensitive to large logits |
