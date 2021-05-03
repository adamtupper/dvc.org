# Checkpoints

ML checkpoints are an important part of deep learning because ML engineers like
to save the model files at certain points during a training process.

With DVC experiments and checkpoints, you can:

- Implement the best practice in deep learning to save your model weights as
  checkpoints.
- Track all code and data changes corresponded to the checkpoints.
- See when metrics start diverging and revert to the optimal checkpoint.
- Automate the process of tracking every training epoch.

[The way checkpoints are implemented by DVC](/blog/experiment-refs) utilizes
_ephemeral_ experiment commits and experiment branches within DVC. They are
created using the metadata from experiments and are tracked with the `exps`
custom Git reference.

You can add experiments to your Git history by committing the experiment you
want to track, which you'll see later in this tutorial.

## Setting up the project

This tutorial is going to cover how to implement checkpoints in a ML project
using DVC. We're going to train a model to identify handwritten digits based on
the MNIST dataset.

You can follow along with the steps here or you can clone the repo directly from
GitHub and play with it. To clone the repo, run the following commands.

```bash
git clone https://github.com/iterative/checkpoints-tutorial
cd checkpoints-tutorial
```

It is highly recommended you create a virtual environment for this example. You
can do that by running:

```bash
python3 -m venv .venv
```

Once your virtual environment is installed, you can start it up with one of the
following commands.

- On Mac/Linux: `source .venv/bin/activate`
- On Windows: `.\.venv\Scripts\activate`

Once you have your environment set up, you can install the dependencies by
running:

```bash
pip install -r requirements.txt
```

This will download all of the packages you need to run the example. Now you have
everything you need to get started with experiments and checkpoints.

## Setting up a DVC pipeline

DVC versions data and it also can version the machine learning model weights
file as checkpoints during the training process. To enable this, you will need
to set up a DVC pipeline to train your model.

Adding a DVC pipline only takes a few commands. At the root of the project, run:

```bash
dvc init
```

This sets up the files you need for your DVC pipeline to work.

Now we need to add a stage for training our model within a DVC pipeliene. We'll
do that with `dvc stage add`, which we'll explain more later. For now, run the
following command:

```bash
dvc stage add --name train --deps data/MNIST --deps train.py \
              --checkpoints model.pt --plots-no-cache predictions.json \
              --params seed,lr,weight_decay --live dvclive python train.py
```

The checkpoints need to be enabled in DVC at the pipeline level. The
`-c / --checkpoint` option of the `dvc stage add` command defines the checkpoint
file or directory. The checkpoint file, _model.pt_, is an output from one
checkpoint that becomes a dependency for the next checkpoint, such as the model
weights file.

The rest of the `dvc stage add` command sets up our dependencies for running the
training code, which parameters we want to track (which are defined in the
_params.yaml_), some configurations for our plots, showing the training metrics,
and specifying where the logs produced by the training process will go.

After running the command above to setup your _train_ stage, your _dvc.yaml_
should have the following code.

```yaml
stages:
  train:
    cmd: python train.py
    deps:
      - data/MNIST
      - train.py
    params:
      - lr
      - seed
      - weight_decay
    outs:
      - model.pt:
          checkpoint: true
    plots:
      - predictions.json:
          cache: false
    live:
      dvclive:
        summary: true
        html: true
```

Before we go any further, this is a great point to add these changes to your Git
history. You can do that with the following commands:

```bash
git add .
git commit -m "created DVC pipeline"
```

Now that you know how to enable checkpoints in a DVC pipeline, let's move on to
setting up checkpoints in your code.

## Registering checkpoints in your code

Take a look at the _train.py_ file and you'll see how we train a convolutional
neural network to classify handwritten digits. The main area of this code most
relevant to checkpoints is when we iterate over the training epochs.

This is where DVC will be able to track our metrics over time and where we will
add our checkpoints to give us the points in time with our model that we can
switch between.

Now we need to enable checkpoints at the code level. We are interested in
tracking the metrics along with each checkpoint, so we'll need to add a few
lines of code.

In the `train.py` file, import the `dvclive` package with the other imports:

```bash
import dvclive
```

Then update the following lines of code in the `main` method inside of the
training epoch loop.

```diff
# Iterate over training epochs.
for i in range(1, EPOCHS+1):
    # Train in batches.
    train_loader = torch.utils.data.DataLoader(
            dataset=list(zip(x_train, y_train)),
            batch_size=512,
            shuffle=True)
    for x_batch, y_batch in train_loader:
        train(model, x_batch, y_batch, params["lr"], params["weight_decay"])
    torch.save(model.state_dict(), "model.pt")
    # Evaluate and checkpoint.
    metrics = evaluate(model, x_test, y_test)
    for k, v in metrics.items():
        print('Epoch %s: %s=%s'%(i, k, v))
+       dvclive.log(k, v)
+   dvclive.next_step()
```

The line `torch.save(model.state_dict(), "model.pt")` updates the checkpoint
file.

The `dvclive.log(k, v)` line stores the metric _k_ with a value _v_ in plain
text files in the _dvclive_ directory by default.

The `dvclive.next_step()` line tells DVC that it can take a snapshot of the
entire workspace and version it with Git. It's important that with this approach
only code with metadata is versioned in Git (as an ephemeral commit), while the
actual model weight file will be stored in the DVC data cache.

## Running experiments

With checkpoints enabled and working in our code, let's run the experiment. You
can run an experiment with the following command:

```bash
dvc exp run
```

You'll see output similar to this in your terminal while the training process is
going on.

```dvc
Epoch 1: loss=1.9428282976150513
Epoch 1: acc=0.5715
Generating lock file 'dvc.lock'
Updating lock file 'dvc.lock'
Checkpoint experiment iteration '4f6044f'.

file:///Users/milecia/Repos/checkpoints-tutorial/dvclive.html
Epoch 2: loss=1.25374174118042
Epoch 2: acc=0.7738
Updating lock file 'dvc.lock'
Checkpoint experiment iteration 'd402c8d'.

file:///Users/milecia/Repos/checkpoints-tutorial/dvclive.html
Epoch 3: loss=0.7242147922515869
Epoch 3: acc=0.8284
Updating lock file 'dvc.lock'
Checkpoint experiment iteration '97e6898'.

file:///Users/milecia/Repos/checkpoints-tutorial/dvclive.html
Epoch 4: loss=0.5083536505699158
Epoch 4: acc=0.8538
Updating lock file 'dvc.lock'
Checkpoint experiment iteration '59ba304'.

file:///Users/milecia/Repos/checkpoints-tutorial/dvclive.html
Epoch 5: loss=0.416655033826828
Epoch 5: acc=0.8777
Updating lock file 'dvc.lock'
Checkpoint experiment iteration '3803071'.

file:///Users/milecia/Repos/checkpoints-tutorial/dvclive.html
Epoch 6: loss=0.36601492762565613
Epoch 6: acc=0.8943
Updating lock file 'dvc.lock'
Checkpoint experiment iteration 'b49dcf8'.

file:///Users/milecia/Repos/checkpoints-tutorial/dvclive.html
Epoch 7: loss=0.3324562609195709
Epoch 7: acc=0.9044
Updating lock file 'dvc.lock'
Checkpoint experiment iteration '03cb7a7'.
```

After a few epochs have completed, stop the training process with `Ctrl + C`.
Now it's time to take a look at the metrics we're working with.

_If you don't have a number of training epochs defined and you don't terminate
the process, the experiment will run for 100 epochs._

## Viewing checkpoints

You can see a table of your experiments and checkpoints in the terminal by
running:

```bash
dvc exp show
```

```dvc
┏━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━┳━━━━━━━━━┳━━━━━━━━┳━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━┓
┃ Experiment    ┃ Created  ┃ step ┃    loss ┃    acc ┃ seed   ┃ lr     ┃ weight_decay ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━╇━━━━━━━━━╇━━━━━━━━╇━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━┩
│ workspace     │ -        │    6 │ 0.33246 │ 0.9044 │ 473987 │ 0.0001 │ 0            │
│ main          │ 02:34 PM │    - │       - │      - │ 473987 │ 0.0001 │ 0            │
│ │ ╓ exp-1db77 │ 02:57 PM │    6 │ 0.33246 │ 0.9044 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ b49dcf8   │ 02:57 PM │    5 │ 0.36601 │ 0.8943 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ 3803071   │ 02:57 PM │    4 │ 0.41666 │ 0.8777 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ 59ba304   │ 02:56 PM │    3 │ 0.50835 │ 0.8538 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ 97e6898   │ 02:56 PM │    2 │ 0.72421 │ 0.8284 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ d402c8d   │ 02:56 PM │    1 │  1.2537 │ 0.7738 │ 473987 │ 0.0001 │ 0            │
│ ├─╨ 4f6044f   │ 02:56 PM │    0 │  1.9428 │ 0.5715 │ 473987 │ 0.0001 │ 0            │
└───────────────┴──────────┴──────┴─────────┴────────┴────────┴────────┴──────────────┘
```

## Starting from an existing checkpoint

Since you have all of these different checkpoints, you might want to resume
training from a particular one. For example, maybe your accuracy started
decreasing at a certain checkpoint and you want to make some changes to fix
that.

First, we need to apply the checkpoint we want to begin our new experiment from.
To do that, run the following command:

```bash
dvc exp apply d402c8d
```

where _d402c8d_ is the id of the checkpoint you want to reference.

Next, we'll change the learning rate set in the _params.yaml_ to `0.000001` and
start a new experiment based on an existing checkpoint with the following
command:

```bash
dvc exp run --set-param lr=0.000001
```

You'll be able to see where the experiment starts from the existing checkpoint
by running:

```bash
dvc exp show
```

You should seem something similar to this in your terminal.

```dvc
┏━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━┳━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━┓
┃ Experiment            ┃ Created      ┃ step ┃     loss ┃    acc ┃ seed   ┃ lr     ┃ weight_decay ┃
┡━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━╇━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━┩
│ workspace             │ -            │   18 │ 0.12399  │ 0.9625 │ 473987 │ 0.001  │ 0            │
│ main                  │ Apr 27, 2021 │    - │       -  │      - │ 473987 │ 0.0001 │ 0            │
│ │ ╓ exp-63287         │ 04:37 PM     │   18 │ 0.12399  │ 0.9625 │ 473987 │ 0.001  │ 0            │
│ │ ╟ 868c144           │ 04:37 PM     │   17 │ 0.12042  │ 0.9616 │ 473987 │ 0.001  │ 0            │
│ │ ╟ 4de4e69           │ 04:37 PM     │   16 │ 0.13808  │ 0.9585 │ 473987 │ 0.001  │ 0            │
│ │ ╟ d8d633c           │ 04:37 PM     │   15 │ 0.20415  │ 0.9397 │ 473987 │ 0.001  │ 0            │
│ │ ╟ 8fbab93 (b3de55f) │ 04:36 PM     │   14 │ 0.24509  │ 0.9309 │ 473987 │ 0.001  │ 0            │
│ │ ╓ exp-4de2a         │ 05:03 PM     │    7 │  0.3096  │ 0.9092 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ d50c724           │ 05:03 PM     │    6 │ 0.33246  │ 0.9044 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ 29491a9           │ 05:02 PM     │    5 │ 0.36601  │ 0.8943 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ b3de55f           │ 05:02 PM     │    4 │ 0.41666  │ 0.8777 │ 473987 │ 0.0001 │ 0            │
```

The existing checkpoint is referenced at the beginning of the new experiment.
The new experiment is referred to as a modified experiment because it's taking
existing data and using it as the starting point.

### Metrics diff

When you've run all the experiments you want to and you are ready to compare
metrics between checkpoints, you can run the command:

```bash
dvc metrics diff 8fbab93 868c144
```

Make sure that you replace `8fbab93` and `868c144` with checkpoint ids from your
table with the checkpoints you want to compare. You'll see something similar to
this in your terminal.

```dvc
Path          Metric    Old      New      Change
dvclive.json  acc       0.9854   0.9831   -0.0023
dvclive.json  loss      0.04828  0.05095  0.00267
dvclive.json  step      19       15       -4
```

_These are the same numbers you see in the metrics table, just in a different
format._

### Looking at plots

You also have the option to generate plots to visualize the metrics about your
training epochs. Running:

```bash
dvc plots diff 8fbab93 868c144
```

where `8fbab93` and `868c144` are the checkpoint ids you want to compare, will
generate a `plots.html` file that you can open in a browser and it will display
plots for you similar to the ones below.

![Plots generated from running experiments on MNIST dataset using DVC](/img/checkpoints_plots.png)
_The results here won't show anything interesting because the model increases in
accuracy so fast._

## Resetting checkpoints

Usually when you start training a model, you won't begin with accuracy this
high. There might be a time when you want to remove all of the existing
checkpoints to start the training from scratch. You can reset your checkpoints
with the following command:

```bash
dvc exp run --reset
```

This resets all of the existing checkpoints and re-runs the code to generate a
new set of checkpoints under a new experiment branch.

```dvc
┏━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━┳━━━━━━━━━┳━━━━━━━━┳━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━┓
┃ Experiment            ┃ Created  ┃ step ┃    loss ┃    acc ┃ seed   ┃ lr     ┃ weight_decay ┃
┡━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━╇━━━━━━━━━╇━━━━━━━━╇━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━┩
│ workspace             │ -        │    4 │ 0.41666 │ 0.8777 │ 473987 │ 0.0001 │ 0            │
│ main                  │ 04:49 PM │    - │       - │      - │ 473987 │ 0.001  │ 0            │
│ │ ╓ exp-a75bb         │ 05:18 PM │    4 │ 0.41666 │ 0.8777 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ b8a4ceb           │ 05:18 PM │    3 │ 0.50835 │ 0.8538 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ 7b652b1           │ 05:17 PM │    2 │ 0.72421 │ 0.8284 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ cf0168f           │ 05:17 PM │    1 │  1.2537 │ 0.7738 │ 473987 │ 0.0001 │ 0            │
│ ├─╨ 8c7f40f           │ 05:17 PM │    0 │  1.9428 │ 0.5715 │ 473987 │ 0.0001 │ 0            │
│ │ ╓ exp-7fafd         │ 05:10 PM │    5 │ 0.36601 │ 0.8943 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ 9ac9e6a           │ 05:09 PM │    4 │ 0.41666 │ 0.8777 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ b869efe           │ 05:09 PM │    3 │ 0.50835 │ 0.8538 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ 422395e           │ 05:09 PM │    2 │ 0.72421 │ 0.8284 │ 473987 │ 0.0001 │ 0            │
│ │ ╟ 44e44a9           │ 05:09 PM │    1 │  1.2537 │ 0.7738 │ 473987 │ 0.0001 │ 0            │
│ ├─╨ c6e11b4           │ 05:09 PM │    0 │  1.9428 │ 0.5715 │ 473987 │ 0.0001 │ 0            │
```

## Adding checkpoints to Git

When you terminate training, you'll see a few commands in the terminal that will
allow you to add these changes to Git.

```dvc
To track the changes with git, run:

        git add dvclive.json dvc.yaml .gitignore train.py dvc.lock

Reproduced experiment(s): exp-263da
Experiment results have been applied to your workspace.

To promote an experiment to a Git branch run:

        dvc exp branch <exp>
```

You can run the following command to save your experiments to the Git history.

```bash
git add dvclive.json dvc.yaml .gitignore train.py dvc.lock
```

You can take a look at what will be commited to your Git history by running:

```bash
git status
```

You should see something similar to this in your terminal.

```git
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   .gitignore
        new file:   dvc.lock
        new file:   dvclive.json

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        data/
        dvclive.html
        dvclive/
        model.pt
        plots.html
        predictions.json
```

All that's left is to commit these changes with the following command:

```git
git commit -m 'saved files from experiment'
```

We can also remove all of the experiments we don't promote to our Git workspace
with the following command:

```bash
dvc exp gc --workspace --force
```

Now that you know how to use checkpoints in DVC, you'll be able to resume
training from different checkpoints to try out new hyperparameters or code and
you'll be able to track all of the changes you make while trying to create the
best possible model.
