![#PyangBind](http://rob.sh/img/pyangbind.png)

**PyangBind** is a plugin for [Pyang](https://github.com/mbj4668/pyang) which converts YANG data models into a Python class hierarchy, such that Python can be used to manipulate data which conforms with a YANG data model.

**PyangBind** does not exhaustively support all YANG data types - and is rather focused to support those that are used within commonly used standard, and vendor-specific models. Contributions to improve YANG (RFC6020) coverage are always welcome!

## Usage

**PyangBind** can be called by using the Pyang ```--plugindir``` argument:

```
pyang --plugindir /path/to/repo -f pybind <yang-files>
```

All output is written to ```stdout``` or the file handle specified with ```-o```.

Once bindings have been generated, each YANG module is included as a top-level class within the output bindings file. Using the following module as an example:

```javascript
module pyangbind-example {
    yang-version "1";
    namespace "http://rob.sh/pyangbind/example";
    prefix "example";
    organization "pyangbind";
    contact "pyangbind@rob.sh";

    description
        "A test module that checks that the plugin works OK";
    revision 2015-04-30 {
        description "Initial Release";
        reference "release.00";
    }

    container parent {
        description
            "A container";

        leaf integer {
            type int8;
            description
              "A test leaf for uint8";
        }
    }
}
```

The module is compiled using ```pyang --plugindir <pyangbind-dir> -f pybind pyangbind-example.yang -o bindings.py``` from the command-line. A consuming application imports the Python module that is output, and can instantiate a new copy of the base YANG module class:

```python
#!/usr/bin/env python

from bindings import pyangbind_example

bindings = pyangbind_example()
bindings.parent.integer = 12
```

Each data leaf can be referred to via the path described in the YANG module. Each leaf is represented by the base class that is described in the [type support](#type-support) section of this document.

Where nodes of the data model's tree are not Python-safe names - for example, ```global``` or ```as```,  an underscore ("\_") is appended to the data node's name. Additionally, where characters that are considered operators are included in the node's name, these are translated to underscores (e.g., "-" becomes "\_").

### YANG Dependencies
With some outputs (e.g., ```tree``` or ```jstree```), Pyang does not require the ability to resolve all typedefs (e.g., it can just display that the type is inet:ip-address). PyangBind **requires** the ability to resolve all typedefs to their underlying definition, such that a new datatype can be defined. In some cases, this will lead to needing to include 'types' YANG files on the command line, such that pyang can load them - and they are made available to PyangBind to build types.

For example; openconfig-bgp will require the path to ietf-yang-types to be specified via:

```
/usr/local/bin/pyang --plugindir ~/Code/pyangbind -f pybind \
-o bindings.py -p ../policy/ bgp.yang ~/path/to/ietf-yang-types.yang 
```
The error messages when a type is not correctly resolved are not very pretty currently and will be made more so in the future.

### Generic Wrapper Class Methods
Each native type is wrapped in a YANGDynClass dynamic type - which provides helper methods for YANG leaves:

* ```default()``` - returns the default value of the YANG leaf.
* ```changed()``` - returns ```True``` if the value has been changed from the default, ```False``` otherwise.
* ```yang_name()``` - returns the YANG data element's name (since not all data elements name are safe to be used in Python).

### YANG Container Methods

Each YANG container is represented as an individual ```class``` within the hierarchy, wrapped with the generic wrappers described above.

In addition, a YANG container provides a set of methods to determine properties of the container:

 * ```elements()``` - which provides a list of the elements of the YANG-described tree which branch from this container.
 * ```get(filter=False)``` - providing a means to return a Python dictionary hiearchy showing the tree branching from the container node. Where the ```filter``` argument is set to ```True``` only elements that have changed from their default values due to manipulation of the model are returned - when filter is not specified the default values of all leaves are returned.

### YANG List Methods

The special ```add()``` and ```delete()``` methods can be used to create and remove entries from the list. Utilising the ```add()``` method will also set the associated key values.

Since YANG lists are essentially keyed - as per Python dictionaries - a ```keys()``` method is provided to retrieve list members. In addition, iterations over a YANGList behave as would be expected from a dictionary in Python (rather than a list).

### YANG String 'pattern' Restrictions

Where a ```pattern``` restriction is specified in the definition of a string leaf, the Python ```re``` library is used to compile this regular expression. As per the definition in RFC6020, ```pattern``` regexps are assumed to be compatible with the definition in [XSD-TYPES](http://www.w3.org/TR/2004/REC-xmlschema-2-20041028/). To this end, all regexps provided have "^" prepended and "$" appended to them if not already specified. This behaviour is different to the default behaviour of ```re.match()``` which will usually allow a user to match a partial string.

### Errors thrown by PyangBind

A number of the errors that PyangBind raises are based on the YANG model input - and are thrown from ```YANGDynClass```. In general, these will be raised when initiating an instance of the generated class (rather than during the initial generation of the bindings) -- in general, PyangBind expects Pyang's validation to have thrown errors *before* the model is parsed (it remains to be seen whether this is a safe assumption!).

Otherwise, PyangBind's generated classes try to be consistent with the error type that a Python programmer would expect from the native types Python provides. That is to say:

* ```KeyError``` is raised when a ```__getitem__``` call results in a member not being found in the referenced item, this includes where members of a YANG ```list``` are not found.
* ```ValueError``` is raised when a value does not conform with the YANG datatype. Since the intention is that a Python programmer does not need to care what types are used 'behind the scenes' by PyangBind ```ValueError``` will only be raised where the input cannot be cast to the  native type (for example, a YANG ```uint32``` will accept ```True``` as an input value, but really store the value of '1' in the leaf).
* ```AttributeError``` will be thrown where a method that does not exist in the encapsulated type is used (as per standard Python errors); a leaf that does not exist in the YANG model is referenced; or an attempt to set a non-configurable value is made. Note that in some cases, ```dir(...)``` on the object may show that the method is available in the YANGDynClass object, since the requirement to capture calls where the data value is changed results in the need to generically overload these methods. 


### Setting ```config: false``` Leaves

```AttributeError``` is also thrown when an attempt to set a ```config: false``` leaf value is made. Backend systems that are populating a model (when reading state from a device for example) should call the ```_set_FOO()``` method directly (which is generated independently of the ```config``` state of the leaf) to set the value.

For example:

```
>>> b.bgp.global_.state.as_ = 12
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: can't set attribute
>>> b.bgp.global_.state._set_as_(12)
>>> b.bgp.global_.state.as_
12
```

## <a anchor="type-support"></a>YANG Type Support

**Type**            | **Sub-Statement**   | **Supported Type**      | **Unit Tests**  
--------------------|--------------------|--------------------------|---------------
**binary**          | -                   | Not supported           | N/A
-                   | length              | Not supported           | N/A
**bits**            | -                   | Not supported           | N/A
-                   | position            | Not supported           | N/A
**boolean**         | -                   | YANGBool                | tests/boolean-empty
**empty**           | -                   | YANGBool                | tests/boolean-empty
**decimal64**       | -                   | [Decimal](https://docs.python.org/2/library/decimal.html) | tests/decimal64 
-                   | fraction-digits     | Supported               | tests/decimal64
**enumeration**     | -                   | Supported               | tests/enumeration
**identityref**     | -                   | Supported               | tests/identityref
**int{8,16,32,64}** | -                   | [numpy int](http://docs.scipy.org/doc/numpy/user/basics.types.html) | tests/int
-                   | range               | Supported               | tests/int
**uint{8,16,32,64}**| -                   | [numpy uint](http://docs.scipy.org/doc/numpy/user/basics.types.html) | tests/int
-                   | range               | Supported               | tests/int    
**leafref**         | -                   | (as string)             | N/A
**string**          | -                   | *str*                   | tests/string
-                   | pattern             | Using python *re.match* | tests/string
-                   | length              | Not supported           | N/A
**typedef**         | -                   | Supported               | tests/typedef
**container**       | -                   | *class*                 | tests/*
**list**            | -                   | YANGList                | tests/list
**leaf-list**       | -                   | TypedList               | tests/leaf-list
**union**           | -                   | Supported               | tests/union
**choice**          | -                   | Supported               | tests/choice

## Licence
             
```
Copyright 2015  Rob Shakir, BT plc. (rob.shakir@bt.com, rjs@rob.sh)

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

## Bugs and Comments

Please send bug-reports or comments to rjs@rob.sh.

## ACKs
* This project was developed as part of BT plc. Network Architecture future network management projects.