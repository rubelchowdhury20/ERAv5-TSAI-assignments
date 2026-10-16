# Session 12

For this assignment I made a small simulator that shows what happens to memory and communication when you train a big model across many GPUs with Data Parallel and then ZeRO-1, ZeRO-2 and ZeRO-3.

The main file is session12_zerosharding_sim.ipynb. It's a notebook, you can run it in colab or jupyter. I didn't use real GPUs, it just pretends there are 32 of them (32 virtual GPUs) and does the arithmetic by hand. The point is to get the numbers, not to actually train anything.

The whole thing runs on one idea which is the 16 bytes per weight. When you train a model normally, each weight ends up costing you 16 bytes: 2 for the weight itself in bf16, 2 for the gradient in bf16, 4 for the fp32 master copy, and then 4 and 4 for Adam's m and v. So weights + gradients + optimizer. That's the full training state. Data Parallel keeps a copy of all of it on every GPU, which is why it eats so much memory.

Then the ZeRO stages keep sharding more and more of that. ZeRO-1 only shards the optimizer state (the biggest chunk, 12 of the 16 bytes). ZeRO-2 also shards the gradients. ZeRO-3 shards everything including the weights. So as you go up the stages, the memory per GPU keeps dropping. In the notebook there's a table that shows bytes per weight and GiB per GPU for each stage, and a bar chart with the H100 memory line drawn on it so you can see which stages actually fit.

That memory saving isn't free. It costs communication. DP, ZeRO-1 and ZeRO-2 all move about 2P per step (P is the parameters in bf16, so 2 times N bytes), and ZeRO-3 moves about 3P because it also has to gather the weights back in the forward and backward pass. The notebook has a table and a plot for that too. What I found interesting is the compute (the FLOPs) stays basically the same no matter which stage you use, sharding just changes where the stuff lives, it doesn't change how much math you have to do.

One thing I want to be clear about, the ZeRO stages only shard the weights, gradients and optimizer state. They don't shard the activations. Activations are a separate thing that depends on the batch and sequence length, and in real training that's handled by sequence parallelism or activation checkpointing, not by ZeRO. I left activations out of the sharding math on purpose and just printed a rough estimate separately so it doesn't get confused with the state.

There's a bit at the end where I check my numbers against the lecture for 30B at G=8 and G=32, just to make sure I didn't mess up the formulas. If you want to run it, open the notebook and run all the cells. If you want to try other sizes just change N_params_billion or G at the top and re-run.

There's also an index.html in here. After doing the notebook I wanted something a bit more friendly to look at, so I turned the same ideas into a little interactive page. The idea behind it is a learning path, it walks you through things a small step at a time and then lets you play with the knobs yourself, so someone who doesn't know any of this can see how the memory gets split and how the GPUs talk to each other. But the notebook is the actual assignment, the html is just extra.
