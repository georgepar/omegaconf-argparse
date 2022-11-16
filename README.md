# omegaconf-argparse
Integration between Omegaconf and argparse for mixed config file and CLI arguments

Flexible configuration management is important during experimentation, e.g. when training machine learning models.

Ideally, we want both a configuration file to hold more "stable" hyperparameter values and the flexibility to
change values through command line arguments for rapid experimentation.

## How it works

This package provides a barebones solution based on the excellent [OmegaConf](https://github.com/omry/omegaconf) package.

Specifically, we extend the `OmegaConf` class with a static `from_argparse` method to parse arguments provided using argparse,
and provide utility functions to merge the default CLI values, YAML configuration file values and user provided CLI arguments.


## Usage

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
* default cli args values < config file values < user provided cli args

E.g.:
* if you don't include a value in your configuration it will take the default value from the argparse arguments
* if you provide a cli arg (e.g. run the script with --bsz 64) it will override the value in the config file


## Generate a configuration file based on an argument parser

Run:

```
from omegacli import generate_config_template
generate_config_template(parser, "example-config.yaml")
```

This will initialize a configuration file, that is consistent with the argument parser.
You can use this as a starting point for saving and editing your configuration.

## Similar solutions

* [Hydra](https://hydra.cc/docs/intro/): A more feature-rich, but more complex solution. If you are willing to introduce the dependency you can use it
* [Pytorch-Lightning](https://pytorch-lightning.readthedocs.io/en/1.6.2/common/lightning_cli.html): PL introduced a similar functionality. You can use it if you are in the PL ecosystem.

## Why create a separate package?

`OmegaConf` plans to remain agnostic to the command line argument parser, therefore we cannot merge this solution upstream. [See related issue](https://github.com/omry/omegaconf/issues/569).
