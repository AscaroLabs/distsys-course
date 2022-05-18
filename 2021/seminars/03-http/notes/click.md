# Click
[Оффициальная документация](https://click.palletsprojects.com/en/8.1.x/)

```python
import click

@click.command()
@click.option('--count', default=1, help='Number of greetings.')
@click.option('--name', prompt='Your name',
              help='The person to greet.')
def hello(count, name):
    """Simple program that greets NAME for a total of COUNT times."""
    for x in range(count):
        click.echo(f"Hello {name}!")

if __name__ == '__main__':
    hello()
```
```sh
$ python hello.py --count=3
Your name: John
Hello John!
Hello John!
Hello John!
```

## Basic Concepts - Creating a Command

```python
import click

@click.command()
def hello():
    click.echo('Hello World!')

if __name__ == '__main__':
    hello()
```
```bash
$ python hello.py
Hello World!
```
```sh
$ python hello.py --help
Usage: hello.py [OPTIONS]

Options:
  --help  Show this message and exit.
```

## Nesting Commands

```python
@click.group()
def cli():
    pass

@click.command()
def initdb():
    click.echo('Initialized the database')

@click.command()
def dropdb():
    click.echo('Dropped the database')

cli.add_command(initdb)
cli.add_command(dropdb)
```

## Adding Parameters

```python
@click.command()
@click.option('--count', default=1, help='number of greetings')
@click.argument('name')
def hello(count, name):
    for x in range(count):
        click.echo(f"Hello {name}!")
```
```sh
$ python hello.py --help
Usage: hello.py [OPTIONS] NAME

Options:
  --count INTEGER  number of greetings
  --help           Show this message and exit.
```

## Options
### Name Your Options
```python
@click.command()
@click.option('-s', '--string-to-echo')
def echo(string_to_echo):
    click.echo(string_to_echo)
```
```python
@click.command()
@click.option('-s', '--string-to-echo', 'string')
def echo(string):
    click.echo(string)
```
> "-f", "--foo-bar", the name is `foo_bar`
> 
> "-x", the name is `x`
> 
> "-f", "--filename", "dest", the name is `dest`
> "--CamelCase", the name is `camelcase`
> 
>"-f", "-fb", the name is `f`
>
>"--f", "--foo-bar", the name is `f`
>
>"---f", the name is `_f`


### Basic Value Options

```python
@click.command()
@click.option('--n', default=1)
def dots(n):
    click.echo('.' * n)
```
```python
# How to make an option required
@click.command()
@click.option('--n', required=True, type=int)
def dots(n):
    click.echo('.' * n)
```
```sh
$ dots --n=2
..
```
```python
# How to use a Python reserved word such as `from` as a parameter
@click.command()
@click.option('--from', '-f', 'from_')
@click.option('--to', '-t')
def reserved_param_name(from_, to):
    click.echo(f"from {from_} to {to}")
```
To show the default values when showing command help, use `show_default=True`
```python
@click.command()
@click.option('--n', default=1, show_default=True)
def dots(n):
    click.echo('.' * n)
```
```sh
$ dots --help
Usage: dots [OPTIONS]

Options:
  --n INTEGER  [default: 1]
  --help       Show this message and exit.
```

### Multi Value Options

```python
@click.command()
@click.option('--pos', nargs=2, type=float)
def findme(pos):
    a, b = pos
    click.echo(f"{a} / {b}")
```
```sh
$ findme --pos 2.0 3.0
2.0 / 3.0
```

### Tuples as Multi Value Options

```python
@click.command()
@click.option('--item', type=(str, int))
def putitem(item):
    name, id = item
    click.echo(f"name={name} id={id}")
```
```sh
$ putitem --item peter 1338
name=peter id=1338
```

### Boolean Flags

```python
import sys

@click.command()
@click.option('--shout/--no-shout', default=False)
def info(shout):
    rv = sys.platform
    if shout:
        rv = rv.upper() + '!!!!111'
    click.echo(rv)
```
```sh
$ info --shout
LINUX!!!!111
$ info --no-shout
linux
$ info
linux
```
If you really don’t want an off-switch, you can just define one and manually inform Click that something is a flag:
```python
import sys

@click.command()
@click.option('--shout', is_flag=True)
def info(shout):
    rv = sys.platform
    if shout:
        rv = rv.upper() + '!!!!111'
    click.echo(rv)
```
```sh
$ info --shout
LINUX!!!!111
$ info
linux
```

### Prompting

```python
@click.command()
@click.option('--name', prompt=True)
def hello(name):
    click.echo(f"Hello {name}!")
```
```sh
$ hello --name=John
Hello John!
$ hello
Name: John
Hello John!
```
If you are not happy with the default prompt string, you can ask for a different one:
```python
@click.command()
@click.option('--name', prompt='Your name please')
def hello(name):
    click.echo(f"Hello {name}!")
```
```sh
$ hello
Your name please: John
Hello John!
```

### Dynamic Defaults for Prompts

The `auto_envvar_prefix` and `default_map` options for the context allow the program to read option values from the environment or a configuration file.

```python
import os

@click.command()
@click.option(
    "--username", prompt=True,
    default=lambda: os.environ.get("USER", "")
)
def hello(username):
    click.echo(f"Hello, {username}!")
```
To describe what the default value will be, set it in `show_default`.
```python
import os

@click.command()
@click.option(
    "--username", prompt=True,
    default=lambda: os.environ.get("USER", ""),
    show_default="current user"
)
def hello(username):
    click.echo(f"Hello, {username}!")
```
```sh
$ hello --help
Usage: hello [OPTIONS]

Options:
  --username TEXT  [default: (current user)]
  --help           Show this message and exit.
```