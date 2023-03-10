# yamlres

*Retrieving algorithm component combinations from online (or local) yaml resources.*

This project extends the *yaml* prototype with web-based resource fields
and creates a declarative algorithm interface. This can help share algorithms
based on existing *Python* libraries via a continuous deployment pipeline
where you only update web resources. Thus, algorithm declarations
are separated from the update cycles of building components.

**Development:** Emmanouil (Manios) Krasanakis<br>
**Dependencies:** `pyyaml`,`wget`

## Loading yaml from the web
`yamlres` parses normal *yaml* files, but also goes through their fields
in search of strings containing the *.yaml* file extension and recursively
replaces such fields from files. Recursive loading throws an exception.
You can reference online web resources to automatically download and parse.

For example, you can download and load an online *yaml* resource with 
the following code:

```python
from yamlres import Loader

resource = "https://raw.githubusercontent.com/maniospas/yamlres/main/examples/ppr.yaml"
specs = Loader().load(resource)
```

You can also access fields within yaml file dicts as if they were loaded objects.
For example, you can access the *import* field of the above file by loading:

```python
print(Loader().load(resource+".import"))
# {'pg': 'pygrank'}
```

<details>
    <summary>Resource accessing within yaml</summary>
Let's see now how accessing a resource can look like from within 
a different yaml file. This example will create a variation
of examples/ppr.yaml (you could also use the respective URL
in place of that file name) that references parts of the latter:

```yaml
import: examples/ppr.yaml.import
assign:
    ranker:
      method: pg.HeatKernel
      args: 3
    posteriors: examples/ppr.yaml.assign.posteriors
return: posteriors
```

</details>

<details>
  <summary>Resource cache</summary>
The above will automatically create a res/ folder at your working 
directory and places downloaded resources in there.
Calling the same resources multiple times will now download them again.
To set a different path to store resources and update them on each run, 
you can call:

```python
specs = Loader(path="yourpath/", update=True).load(...)
```
</details>

## Algorithms from yaml definitions
`yamlres` lets you share definitions of algorithmic pipelines in *yaml* 
format. These should be appropriate dictionaries and can be loaded in 
the form of runnable methods with appropriate keyword arguments per:

```python
from pyyaml import Runner

algorithm = Runner().init(specs)
print(algorithm(kwarg1=..., kwarg2=...))
```

The following dictionary fields are allowed in algorithm definitions:

**definitions**
Place any *yaml* anchor definitions here. No additional processing
takes place for these .

**import**
This is a dictionary from aliases to respective libraries. You can use methods
of imported libraries in your definitions. For example, the expression
`import libraryname as lib` is converted to `yamlres` format to the following
snippet and lets you reference methods with the pattern *lib.methodname* :

```yaml
import:
  lib: libraryname
```

<details>
  <summary>Dependency safety</summary>
Running defined algorithms is safe in that it runs only on dependencies that
end users have declared. To fully constraint runs on a predefined set of 
dependencies, you can pass these as a list argument to the runner,
for example as in the following snippet:

```python
algorithm = Runner(trust=["pygrank"]).init(specs)
```

</details>

**assign**
This is a dictionary of value assignments. If the assignment
is a dictionary and it has a *method* field, then the respective
method is called based on the optional *args* and *kwargs* fields. For
example, to call a method and assign the returned value to a
variable, you can call:

```yaml
assign:
  variablename:
    method: methodname
    args: [arg1, arg2]
    kwargs:
      argname1: argvalue1
      argname2: argvalue2
```

Variables can be used as inputs to other method calls or be returned
at the end. You can also directly define method calls within
arguments or keyword arguments, though you won't be able to 
programmatically override those afterwards.

<details>
    <summary>Functional pipeline</summary>
You can define a functional pipeline by calling the namesake method,
of the yaml.functional module. This starts from an input and calls
each consequent methods

```yaml
import:
  func: yamlres.functional
assign:
  output:
    method: func.pipeline
    args:
      - input
      - methodname
      - method: MethodBuilder1
      - method: MethodBuilder2
```
</details>


<details>
    <summary>Method builders</summary>
You can define a method builder by calling the namesake method,
of the yaml.functional module. This starts from a method and
arguments to override, and fills in the required ones later.
Mostly this is used to generate the equivalents of
lambda expressions for methods that require other methods 
as inputs:

```yaml
import:
  func: yamlres.functional
assign:
  methodvariation:
    method: func.builder
    args: basemethod
    kwargs: 
      kwarg1: value1
      kwarg2: value2
  output:
    method: func.pipeline
    args:
      - input
      - methodvariation  # will call basemethod(input, kwarg1=value1, kwarg2=value2)
      - method: MethodBuilder1
      - method: MethodBuilder2
```
</details>


<details>
  <summary>Programmatically overriding values</summary>
Any arguments you provide to runners override any internal
definitions. For example, you can run the examples/ppr.yaml
file with value alpha=0.9 with the following code snippet:

```python
from yamlres import Loader, Runner

resource = "https://raw.githubusercontent.com/maniospas/yamlres/main/examples/ppr.yaml"
specs = Loader().load(resource)
algorithm = Runner().init(specs)
print(algorithm(priors=..., alpha=0.9))
```

</details>

**return**
This declares either a single value or a dicitionary of values
to return when your defined algorithm is run.

