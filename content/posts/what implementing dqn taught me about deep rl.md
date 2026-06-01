---
title: What Implementing DQN Taught Me About Deep RL
date: 2026-05-22
draft: false
comments: true
math: true
tags:
  - RL
---
# Intro
Why do I want to implement RL algorithm from scratch? After taking so many courses, experiments, exams during Bachlor and Master, I truly realize that purely studying without actually doing, building something with my hand gives me an illusion that I've learned something. Same thing happens when I learn reinforcement learning. I took the course, learned the concepts, passed the exam. I could recite those concepts and definitions word for word, but when I actually tried to solve a real problem or explain it clearly to someone, my mind goes blank. Therefore, I have made up my mind to implement algorithm from scratch diving deep into every technical details. 
After finishing it, it turned out there are many details I would never think of if I didn't do this. I'd like to share my experience in DQN implementation with people who wants to learn RL algo and the version of myself from three months ago.
# Architecture of DQN
DQN(Deep Q-network) is composed of Q-learning and deep neural network. Q-learning is a value-based method using TD learning to train the Q-function. Deep neural network is a foundamental concept from Deep Learning. Before diving into DQN, let's quickly go over these two components.
## Q-Learning
Letter **Q** in Q-learning stands for quality for a state-action pair. Given a specific state, there are many different actions to take based on different environments. We want to determine their quality, which in other word, is **how valuable this action is at this state**. Therefore, Q-learning is basically to estimate a Q-function that represents every possible state-action pair. In practice, this Q-function is encoded by a Q-table. Each cell corresponds to a specific state-action pair value. So when agent is at a state wanting to take an action, it searches inside the Q-table to find the answer. 
<figure>

| State \ Action | Left  | Right | Up    | Down  |
|:-----:|:-----:|:-----:|:-----:|:-----:|
| S0    | -0.12 | **0.85** | 0.32  | -0.05 |
| S1    | 0.44  | -0.30 | **0.91** | 0.27  |
| S2    | 0.66  | 0.51  | -0.18 | **0.78** |
| S3    | 0.39  | **0.97** | 0.62  | -0.08 |

<figcaption>Example Q-table with optimal actions highlighted in bold</figcaption>
</figure>

At the beginning, the Q-table is useless since all the values are just random number, but after training, it becomes **optimal Q-table** which means agent knows which action is the best action to take at that state. 
Then, how to train this Q-table? We can follow the following steps.
1. *Initialization* - Initialize the Q-table. Arbitrarily choose the value of each cell.
2. *Action selection* - Choose the action at the current state according to $\epsilon$ greedy policy( The agent explores the env with probability of $\epsilon$ and exploits what it's learned with probability of $1 - \epsilon$ ).
3. *Take action* - Take chosen action, observe the next state and receive the reward.
4. *Update the Q-table* - Use TD-Learning to update the Q-table. What TD-Learning says is that the agent update the current estimation( current Q-value ) based on the next estimation( next Q-value ). $V(S_{t})\leftarrow V(S_{t})+\alpha[R_{t+1}+\gamma V(S_{t+1})-V(S_{t})]$ Because there are many actions for agent to take, we use greedy policy to choose the highest state-action pair value. Thus, the equation becomes $Q(S_{t},A_{t})\leftarrow Q(S_{t},A_{t})+\alpha[R_{t+1}+\gamma max_{a}Q(S_{t+1},a)-Q(S_{t},A_{t})]$
5. *Repeat step 2-4*

After thousands of episodes, this Q-table is close to the optimal Q-table. The agent knows which action is the best to take at a given state.
## Deep neural network
Neural network is just a function approximator. It maps the input $x$ to output $y$. Deep means the depth of the neural network, in other words, the amount of layers in this neural network. The deeper this neural network is, the more complex patterns it can learn.
$$
y=f(x)
$$
The goal of neural network is to find the best estimate of a target output by adjusting the parameters to the optimal parameters. Specifically, we have following steps to train the neural network.
- Feed a bunch of training data into neural network and get an prediction. 
- Compare the prediction and the ground truth from training data to form loss function( the distance between prediction and ground truth ). 
- Use Backpropagation to calculate gradient of loss function w.r.t. each parameters.
- Gradient descent to update each parameter.
- Obtain the optimal parameters after training.
## Deep Q-network
When we compare Q-learning and deep neural network, we can see the similarities of Q-learning and Neural network. The goal of both is to find an optimal function(Q-function and approximation function). The way to achieve this goal is both to minimize the distance between TD-target/ground truth and current Q-value/prediction. That's why we can integrate deep neural network with Q-learning to create deep Q-network. We use neural network to approximate optimal Q-function.

Thus, in practice, we feed state of an agent into neural network, whether it's pixels or coordinates, they are all numbers as input. The output will be every possible Q-value that agent can have. After training, each state is going to correspond to a set of Q-values and we just choose the largest one.
image: architecture of DQN

