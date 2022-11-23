# omegaconf-argparse

Integration between Omegaconf and argparse for mixed config file and CLI arguments

Flexible configuration management is important during experimentation, e.g. when training machine learning models.

Ideally, we want both a configuration file to hold more "stable" hyperparameter values and the flexibility to
change values through command line arguments for rapid experimentation.

This package provides a barebones solution based on the excellent [OmegaConf](https://github.com/omry/omegaconf) package.

Specifically, we extend the `OmegaConf` class with a static `from_argparse` method to parse arguments provided using argparse,
and provide utility functions to merge the default CLI values, YAML configuration file values and user provided CLI arguments.

# Installation

Install package from PyPi:

```
pip install omegaconf-argparse
```

## Usage

Let's start with an example. You have an argparse based script, and you want to move to a more reproducible setup
based on configuration files, without losing the flexibility of cli arguments.

In `examples/mnist.py` we have modified the example MNIST training script from [pytorch repo](https://github.com/pytorch/examples/blob/main/mnist/main.py).

The diff is:

```
11a12,13
> from omegacli import generate_config_template, parse_config
>
158a161,172
>     parser.add_argument(
>         "--generate-config",
>         action="store_true",
>         default=False,
>         help="Generate example YAML configuration file",
>     )
>     parser.add_argument(
>         "--config-path",
>         type=str,
>         default=None,
>         help="Path to configuration file",
>     )
159a174,182
>
>     if args.generate_config:
>         import sys
>
>         generate_config_template(parser, args.config)
>         sys.exit(0)
>
>     args = parse_config(parser, args.config)
```

Note we have added two command line arguments `--generate-config` and `--config-path`.

When we run

```
python mnist.py --generate-config --config-path config.yaml
```

it will create a `config.yaml` file, which can be used from now on:

```yaml
batch_size: 64
test_batch_size: 1000
epochs: null
lr: 1.0
gamma: 0.7
no_cuda: false
no_mps: false
dry_run: false
seed: 1
log_interval: 10
save_model: false
```

Now if we run:

```
python mnist.py --config config.yaml
```

The script will use the values provided in the `config.yaml` file. If we change the configuration:

```yaml
lr: 0.1
```

The training will use `lr=0.1`.

At any time we can run a quick experiment (let's say with `gamma=1.0`) and override the config values using the CLI:

```
python mnist.py --config config.yaml --gamma 1.0
```

When we are confident with our experiments we can set the best hyperparameter values in the configuration file and push to a remote repo for reproducibility.

## How it works

We provide a high-level utility function `parse_config` that merges a YAML configuration file with an `argparse.ArgumentParser`:

```
import io
from omegacli import parse_config
mock_config_file = io.StringIO('''
model:
  hidden: 100
''')
parser = argparse.ArgumentParser("My cool model")
parser.add_argument("--hidden", dest="model.hidden", type=int, default=20)
cfg = parse_config(parser, mock_config_file)
>>> {'model': {'hidden': 100}}
type(cfg)
>>> <class 'omegaconf.dictconfig.DictConfig'>
cfg = parse_config(parser, mock_config_file, args=["--hidden", "200"])
>>> {'model': {'hidden': 200}}
mock_config_file = io.StringIO('''
random_value: hello
''')
cfg = parse_config(parser, mock_config_file)
>>> {'model': {'hidden': 20}, 'random_value': 'hello'}
```

You can also use the patched `OmegaConf` class directly:

```
import argparse
from omegacli import OmegaConf
parser = argparse.ArgumentParser("My cool model")
parser.add_argument("--hidden", dest="model.hidden", type=int, default=20)
user_provided_args, default_args = OmegaConf.from_argparse(parser, args=["--hidden", "100"])
user_provided_args
>>> {'model': {'hidden': 100}}
default_args
>>> {}
user_provided_args, default_args = OmegaConf.from_argparse(parser)
user_provided_args
>>> {}
default_args
>>> {'model': {'hidden': 20}}
```

**NOTE**: the `from_argparse` method calls the `parser.parse_args()`.

## Merging of provided values

The precedence for merging is as follows

- default cli args values < config file values < user provided cli args

E.g.:

- if you don't include a value in your configuration it will take the default value from the argparse arguments
- if you provide a cli arg (e.g. run the script with --bsz 64) it will override the value in the config file

### Conventions

To create a nested configuration structure and match with the argparse provided CLI args,
we use the `dest` kwarg when adding an argument with `parser.add_argument`.
Specifically, we follow the convention that the destination is a string, delimited by `.` to indicate nested structure.

For example:

```
parser.add_argument("--hidden", dest="model_hidden", type=int, default=20)
```

will create a top-level element in the configuration:

```
user_provided_args, default_args = OmegaConf.from_argparse(parser, args=["--hidden", "100"])
user_provided_args
>>> {'model_hidden': 100}
```

This will match with the following entry in the YAML file:

```
model_hidden: 100
```

The following:

```
parser.add_argument("--hidden", dest="model.hidden", type=int, default=20)
```

will create a nested structure in the configuration:

```
user_provided_args, default_args = OmegaConf.from_argparse(parser, args=["--hidden", "100"])
user_provided_args
>>> {'model': {'hidden': 100}}
```

and will match with the following YAML entry:

```
model:
  hidden: 100
```

The parsing is recursive, so you can go as deep as you want.

## Generate a configuration file based on an argument parser

Run:

```
from omegacli import generate_config_template
generate_config_template(parser, "example-config.yaml")
```

This will initialize a configuration file, that is consistent with the argument parser.
You can use this as a starting point for saving and editing your configuration.

## Similar solutions

- [Hydra](https://hydra.cc/docs/intro/): A more feature-rich, but more complex solution. If you are willing to introduce the dependency you can use it
- [Pytorch-Lightning](https://pytorch-lightning.readthedocs.io/en/1.6.2/common/lightning_cli.html): PL introduced a similar functionality. You can use it if you are in the PL ecosystem.

## Why create a separate package?

`OmegaConf` plans to remain agnostic to the command line argument parser, therefore we cannot merge this solution upstream. [See related issue](https://github.com/omry/omegaconf/issues/569).
