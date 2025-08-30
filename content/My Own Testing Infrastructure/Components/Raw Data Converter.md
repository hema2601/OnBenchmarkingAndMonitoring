---
title: Raw Data Converter
draft: false
tags:
---

The Raw Data Converter is a beautiful little piece of software. It is quite easily extendable in contrast to the hard-coding from the [[Summarizer]].

But what does it do? As its name suggests, the Raw Data Converter takes the raw data files, which are not yet in json format, and - through user defined functions - translates them into a json file, that can then be easily read and further processed by the other components of the infrastructure. It is written in python and is therefore able to do some powerful post-processing that wasn't possible in bash-scripts.

It is based on the `JsonGenerator` class. The idea is that the `JsonGenerator` implements basic functionalities, such as opening the raw file at the beginning and closing the final file in the end, while leaving some functions empty for the user to define. Then, every user-defined Generator for a specific type of data file inherits the `JsonGenerator`, so the user does not have to deal with any boiler-plate code and can just focus on implementing the relevant parts.

The `JsonGenerator` looks like this:

```python
class JsonGenerator:

    #[...]

    def __init__(self, path):
        if os.path.isfile(path) is False:
            raise ValueError('No file with the name {}'.format(path))
            self.f = None
        else:
            self.f = open(path, 'r+')
        self.json_dict = list()
	#[...]

    def generate_json(self):
        print("Implement this in Child class. This is an empty function")

    def read_source(self):
        print("Implement this in Child class. This is an empty function")

    def cleanup(self):
        self.f.close()
```

I omit some parts of the implementation for now, because its a bit complicated and will make more sense later. 

Now, for every kind of data file, we want to create a custom generator. A list of file types can be found at the beginning of the Raw Data Converter.

```python
# Add new filetypes here
class Filetype(Enum):
    PACKET_CNT      = 1
    SOFTIRQ         = 2
    IRQ             = 3
    IPERF           = 4
    SOFTNET         = 5
    PKT_STEER       = 6
    PROC_STAT       = 7
    PERF            = 8
    PERF_STAT       = 9
    BUSY_HISTO      = 10
    IPERF_LAT       = 11
    PKT_LAT_HISTO   = 12
    NETSTAT         = 13
    PKT_SIZE_HISTO  = 14
Filetype = Enum('Filetype', ['PACKET_CNT', 'SOFTIRQ', 'IRQ', 'IPERF', 'SOFTNET', 'PKT_STEER', 'PROC_STAT', 'PERF', 'PERF_STAT', 'BUSY_HISTO', 'IPERF_LAT', 'PKT_LAT_HISTO', 'NETSTAT', 'PKT_SIZE_HISTO'])
```

And then, each file type will have to have an implementation later in the code:

```python

class PACKET_CNTGen(JsonGenerator):
    def generate_json(self):
    # [...]
    def read_source(self):
    # [...]
    
class SOFTIRQGen(JsonGenerator):
    def generate_json(self):
    # [...]
    def read_source(self):
    # [...]


# [...]
```

An explanation of how exactly to add a new data type in the Raw Data Converter is covered [[Adding New Data Sources|here]], so I won't be getting into any detail. Just know that for every data file, a file type and a generator inheriting the `JsonGenerator` class will need to be added to the code.

So how does the Raw Data Converter operate?
It takes two arguments: The name of the target experiment and a list of file types that should be converted. Then, it loops over every given file type, creates the generator, and dispatches the defined functions. This is where the inheritance shines. Since everything is implemented in the parent class, the main can be super generic and the user does not have to touch it at all.

```python
current_path="/home/hema/testing_infrastructure/"
folder = current_path + "data/" + sys.argv[1] + "/"

# Error Handling
# [...]
#====

for target in sys.argv[2:]:

    ftype = Filetype[target]

    gen = JsonGenerator.instantiate(ftype, folder)

    if gen is None:
        continue

    gen.read_source()
    gen.generate_json()
    gen.cleanup()
```

>[!hint]- Possible change to the argument list
>Usually when I run experiments, I want all present data files to be converted. It would be great to add a feature where, if no file type list is given (aka `argc == 2`), the Raw Data Converter automatically converts all present data files it can identify. This should be pretty straight-forward to implement, since the naming of the file type and the actual data file are related (explained [[Adding New Data Sources|here]])

If you are curious, you might wonder what the `instantiate` function does. This is the part I omitted earlier in the `JsonGenerator`. Usually, when creating an instance of a class, you would write something like `gen = JsonGenerator()`. The problem with that is that the custom generators have their own names and if you had to create an instance of them by calling the class constructor explicitly (`gen = SOFTIRQGEN()`), then the user would have to insert even more code and it would get cluttered. Therefore, the `instantiate` function is written such that by giving it a file type, it automatically creates an object of the class \<file type\>Gen. 

```python 
 @staticmethod
    def instantiate(target, path):
        gen = None
        full_path = None
        for ftype in Filetype:
            if(ftype == target):
                gen = __class__._tr.get(ftype.name + "Gen")
                full_path = path + ftype.name.lower() + ".json"

        ret = None
        try:
            ret = gen(full_path)
        except ValueError:
            print("Warning: Failed to instantiate for {}! Skipping...".format(full_path))
            return None
        else:
            return ret

    def __init_subclass__(cls):
        __class__._tr[cls.__name__] = cls
```

I'm not a python person, so I do not understand this fully in-depth, but the gist is that the `__init_subclass__` function is called every time a class is created that inherits `JsonGenerator`. When that happens, we put that class on a list associated with the `JsonGenerator` class called `_tr`. Then, in our `instantiate` method, we check whether a class by the name `<file type>Gen` exists, and if it does we save it into the `gen` variable.
Then we try to create the object by calling the constructor associated with the `gen` object. If it fails, we print a warning , otherwise we return the object. Pretty neat, right?

# The Different Generators

There is unfortunately no time to explain the implementation of the individual generators that are contained in the Raw Data Converter. I go over an example in [[Adding New Data Sources#1.1.1.2 Example of a JsonGenerator implementation|this tutorial]]. I invite you to brush up on your knowledge of json usage in python and read the generators by yourself. After a few, the others will start looking quite alike.