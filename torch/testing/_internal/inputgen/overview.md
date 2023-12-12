# Overview

InputGen's objective is to generate inputs to torch ops, given certain constraints, which specify when some input is valid for the op and when it is not. InputGen consists of such set of specifications for each argument of an op, and an input generation engine.

## Arguments

Before giving an overview of the specifications and the generation engine, let's first examine the argument themselves.

### ArgType

To being describing an argument, we first need to declare its type. An argument might be a Tensor, an Optional Tensor, a List of Tensors, a Scalar. It could also be just a boolean, or a float, or an int, or a list of int, etc. However, there a "different types" of int. For example, an int argument might be a dimension, or could be an index, or could be how much pad to add on each side. There are different constraints imposed on the "int" depending on the case. Therefore, we decided to introduce the ArgType abstraction. Different ArgTypes are: Tensor, Scalar, Dim, DimList, Index, Length, etc.

### MetaArg

When generating an argument, we first generate *everything except the tensor data* (if we are dealing with a tensor or a list of tensors). That "everything except the tensor data" is what we call the meta argument. For an int, or a boolean or a Scalar, the meta argument contains all of the same information that the argument contains, but organized a bit differently.

A meta argument contains the *structure* of the argument (think about rank, and sizes in the case of a tensor, or the length of a list of integers). It also contains the dtype of the argument (if talking about tensors or scalars). It also contains the value of the argument (if talking about anything except a tensor or list of tensors)

### Attributes

Arguments have attributes. A tensor has dtype, rank, sizes and values. A list of integers has length and values. A scalar has dtype and value. A list of tensors has length, dtypes, ranks, sizes and values. We have consolidated argument qualities behind 6 attributes: Optional, Dtype, Length, Rank, Size, Value.
(Optional is a boolean specifying if the argument is null or not.)

## Specifications

An argument of an op might depend on another argument. For example, an argument specifying a dimension of a tensor depends on the tensor's rank. For this reason, the specifications for an argument consist of a list of dependencies (other arguments) and a list of constraints (based on those dependencies). The constraints are given at the attribute level. For example, when generating a list of tensors, we need to know the valid lengths, and given that length, the valid ranks, and dtypes, and then the valid sizes, and values.

### Constraints

A given constraint has 3 parts: an attribute, a suffix and a lambda function.
We already went through the attributes. The suffix is a basic operation, like Equal (Eq), Not Equal (Ne), Less than (Lt), Greather than (Gt), etc.
The lambda function takes the dependencies (other arguments) as inputs, and outputs certain value. The constraint should then be read as: attribute suffix value. For example, if the attribute is Length, and suffix is Gt, and the lambda function outputs 4, then the constraint is saying that the Length > 4.

## Generation

Now, let's briefly describe the key concepts involved in our generation process.

### Variable

To make an analogy, if the universe is made of inputs to torch ops, then a variable is a fundamental particle. They are the building blocks for generating more complex structures.

### Variable Types

We currently have 7 variable types:
 - Bool (True or False)
 - Int (any integral value)
 - Float (any floating point value)
 - String (any string)
 - ScalarDtype (currently this can take only 3 values: bool, int or float)
 - TensorDtype (torch.dtype)
 - Tuple (any tuple of values)

A Length attribute is generated using a variable of type Int. So is a Rank attribute. On the other hand a Dtype attribute depends on the ArgType. If the ArgType is Scalar, then the Dtype attribute is generated using a variable of type ScalarDtype. If the ArgType is Tensor, then it is generated using a variable of type TensorDtype. A Value attribute is even more complex: it depends on the ArgType of course, since the Value of a Dim argument is int, while the Value of a Bool argument is bool. However, if the ArgType is Scalar, then the Value attribute also depends on the dtype of the scalar (bool, int or float).

### Solvable Variable

The important point to keep in mind is that we generate everything using variables. Therefore, the constraints imposed on the argument attributes end up trickling down these variables. Therefore our variables need to be solvable, i.e. they need to be able to handle constraints similar to the ones imposed on the attributes. Each constraint modifies the space of values that the variable can take.

### Variable Space

A variable space needs to be initialized with a variable type. Its purpose is to describe the space of values that a given variable can take. This space can be discrete (like a Python set) or it can be a union of disjoint intervals of real numbers (only for suitable variable types).

### Variable Generator

A variable generator needs to be initialized with a variable space. It does the job of generating values from that variable space.

### Engine

What is an engine? It is something that has the ability to solve constraints and generate values according to those constraints.

### Attribute Engine

Responsible for producing attributes.

### Structural Engine

Responsible for producing the structure of an argument.

### Meta Argument Engine

Responsible for producing meta arguments.

### Meta Tuple Engine

Responsible for producing tuples of meta arguments.

### Argument Generator

Once we have produced a meta argument, there is nothing else we need to solve. We only need to generate. Therefore, we don't have an Argument Engine, instead we have an Argument Generator. This object needs to be initialized with a Meta Argument, and it produces the actual values to the argument, ready to be given to the torch op as input.