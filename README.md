# Introduction

Offline policy evaluation (OPE) is an active area of research in reinforcement learning. The aim, in a contextual bandit setting, is to take bandit data generated by some policy (let's call it the production policy) and evaluate a new candidate policy offline. The use case is clear: before you deploy a new policy, you want to estimate its performance, and compare to what's already deployed.

In supervised learning settings, the standard approach to offline evaluation is to train on a train set and estimate generalisation performance on a holdout set. In online learning settings, one typically uses progressive validation. In contextual bandit settings, neither is directly possible, because like all reinforcement learning, there is a partial information problem: you never get to see rewards of actions you didn't take. Your only source of information is the bandit data generated by your production policy, which might make entirely different choices than your candidate policy.

It doesn't, then, seem possible to reliably evaluate contextual bandit policies offline. But it is! The key is to use estimators that fill in fake rewards for actions that weren't taken, thereby creating a "fake" supervised learning dataset, against which you can estimate performance, either using progressive validation or a holdout set.

VW implements several estimators to reduce policy evaluation to supervised learning-type evaluation. The simplest method, the direct method (DM) simply trains a regression model that estimates the cost (negative reward) of an (action, context) pair. As you might suspect, this method is generally biased, because the partial information problem means you typically see many more rewards for good actions than bad ones (assuming your production policy is working normally). Biased estimators should not be used for offline policy evaluation, but VW implements provably unbiased estimators like inverse propensity weighting (IPS) and doubly robust (DR) that can be used for this purpose.

Finally, before we get into how to run offline policy evaluation in VW, note that in this tutorial, by policies we mean contextual bandit models, not the exploration layer (e.g. epsilon-greedy) that is usually part of a contextual bandit system to tackle the explore-exploit tradeoff where we must try different actions to learn what works and what doesn't. 

For now, If you wish to evaluate the performance of the entire loop (model + exploration), please refer to the documentation for `--explore_eval`. It is useful if you want to understand how different types of exploration might lead to better future rewards in an online learning bandit system.

## Policy evaluation with `cb`-format data, using a premade policy

If your production policy produces bandit data in the standard `cb` format, and you already have a candidate policy, you can use the `--eval` option to perform OPE. Note that your candidate policy doesn't need to be trained using VW.

First, create a new file, e.g. `eval.dat`. Then, for each instance of your bandit data, write it to `eval.dat` but prepend the line with the action your candidate policy would have chosen given the same context. For example, if your current instance is `1:2:0.5 | feature_a feature_b` and your candidate policy chooses action 2 instead given the same context `feature_a feature_b`, write the line `2 1:2:0.5 | feature_a feature_b` (note the space!). 

After you've written your data file, it might look something like this:
```
2 1:2:0.5 | feature_a feature_b
2 2:2:0.4 | feature_a feature_c
1 1:2:0.1 | feature_b feature_c
```
In the toy example above, the candidate agreed with the production policy for the second and third instances, but disagreed on the first instance.

You are now ready to run policy evaluation using the command `vw --cb <number_of_arms> --eval -d <dataset>`. In our example, we have two possible actions, so the command is `vw --cb 2 --eval -d eval.dat`. This produced the following output (your results might differ based on VW version, or the seed):

    Num weight bits = 18
    learning rate = 0.5
    initial_t = 0
    power_t = 0.5
    using no cache
    Reading datafile = eval.dat
    num sources = 1
    Enabled reductions: gd, scorer, csoaa, cb
    average since     example    example current current current
    loss   last     counter     weight  label predict features
    0.000000 0.000000      1      1.0  known    2    3
    2.500000 5.000000      2      2.0  known    2    3

    finished run
    number of examples = 3
    weighted example sum = 3.000000
    weighted label sum = 0.000000
    average loss = 6.501957
    total feature number = 9

The key metric is `average loss`. How it is calculated depends on the estimator; for example, if you wish to use an estimator other than the default `DR` (TODO confirm that this is the default), you may do so using the `--cb_type` option. For IPS, run `vw --cb 2 --eval -d eval.dat --cb_type ips`:

    Num weight bits = 18
    learning rate = 0.5
    initial_t = 0
    power_t = 0.5
    using no cache
    Reading datafile = eval.dat
    num sources = 1
    Enabled reductions: gd, scorer, csoaa, cb
    average  since         example        example  current  current  current
    loss     last          counter         weight    label  predict features
    0.000000 0.000000            1            1.0    known        2        3
    2.500000 5.000000            2            2.0    known        2        3

    finished run
    number of examples = 3
    weighted example sum = 3.000000
    weighted label sum = 0.000000
    average loss = 8.333333
    total feature number = 9
    
This toy example has far to few examples to form a reliable estimate of the candidate policy's performance, but generally, the `average loss` estimate if the OPE estimate.

The interpretation of OPE is important to get right: if your candidate policy produces an OPE estimate of `3.0` and your production policy has an average loss of `6.0` (easily calculated by summing the costs in the bandit data, divided by the number of instances), it means that had you deployed the candidate policy, with no exploration, instead of the production policy _at the time the production policy was deployed_, you could have expected to se average costs reduce by 50%.

Finally, note what happens if we try to run `--eval` with an estimator we know is biased, `vw --cb 2 --eval -d eval.dat --cb_type dm`. You will end up with an error, to prevent you from making a mistake:

    Error: direct method can not be used for evaluation --- it is biased.

    finished run
    number of examples = 0
    weighted example sum = 0.000000
    weighted label sum = 0.000000
    average loss = n.a.
    total feature number = 0
    direct method can not be used for evaluation --- it is biased.
    vw (cb_algs.cc:161): direct method can not be used for evaluation --- it is biased.
