# Deep Learning Roadmap

A staged path from foundations to building and deploying real neural networks. Move on only when you can *build* the thing, not just read about it. (Rough timeline assumes ~8–12 focused hours/week.)

## Phase 0 · Prerequisites 
Goal: get comfortable enough with math and Python that the DL parts aren't fighting your tooling.

- Math you need: linear algebra (vectors, matrices, matrix multiplication), calculus (derivatives, chain rule, gradients), probability (distributions, conditional probability, expectation). Intuition over proofs.
- Tools: NumPy (broadcasting, vectorization), Pandas, Matplotlib, Jupyter/Colab (free GPUs), Git.
- Checkpoint: implement gradient descent for linear regression in pure NumPy.

## Phase 1 · Neural Network Core 
Goal: understand what a network is and how it learns.

- Concepts: the neuron, layers, forward propagation, activations (ReLU, sigmoid, tanh, softmax), losses (MSE, cross-entropy), backpropagation, optimizers (SGD, momentum, Adam).
- Training well: overfitting vs underfitting, train/val/test splits, dropout, weight decay, early stopping, batch norm, tuning learning rate/batch size/epochs.
- Framework: PyTorch (recommended) or TensorFlow.
- Project: build a feed-forward net from scratch in NumPy, then rebuild in PyTorch and classify MNIST.

## Phase 2 · Vision & Sequences 
Goal: the two classic architecture families — CNNs for images, RNNs for ordered data.

- CNNs: convolutions, filters, pooling, feature maps; LeNet/AlexNet/VGG/ResNet; transfer learning & fine-tuning; data augmentation.
- Sequences: RNNs and their limits, LSTM & GRU, word embeddings (word2vec, GloVe), seq2seq and the attention mechanism.
- Projects: fine-tuned ResNet image classifier; text sentiment classifier or character-level text generator.

## Phase 3 · Transformers & Modern DL 
Goal: where the field is now — attention replaced recurrence.

- The Transformer: self-attention, multi-head attention, positional encoding, encoder/decoder blocks, "Attention Is All You Need", BERT vs GPT families.
- Applying large models: Hugging Face Transformers, pretraining vs fine-tuning, LoRA/PEFT, prompting, embeddings, RAG basics; generative models (VAEs, GANs, diffusion) overview.
- Project: fine-tune a small pretrained transformer on a custom dataset using Hugging Face.

## Phase 4 · Production & Depth (ongoing)
Goal: turn models into things people use, and specialize.

- Ship it: save/load models, serve with FastAPI or a cloud endpoint, GPU training & mixed precision, experiment tracking (W&B, TensorBoard), MLOps (versioning, monitoring, reproducibility).
- Go deeper: pick a track (NLP/LLMs, vision, generative AI), read papers weekly (Papers with Code, arXiv), reproduce a paper, compete on Kaggle, contribute to open source.
- Capstone: ship one end-to-end project — data → trained model → deployed API/app — and write it up.

## How to actually get through this
- Build before you read the next thing.
- Do the math (backprop) by hand once, then let PyTorch do it forever.
- Use free GPUs (Colab, Kaggle Notebooks).
- Keep a project portfolio — every phase ends with something on GitHub.
- Don't chase every new model; fundamentals transfer.

## Free resources
Andrej Karpathy's "Neural Networks: Zero to Hero", fast.ai Practical Deep Learning, DeepLearning.AI specializations, Dive into Deep Learning (d2l.ai), PyTorch official tutorials.
