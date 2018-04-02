# Multiple Outputs from a Single Model

In many cases it is desired to generate multiple independent outputs from the same input model. For example XML descriptions of interfaces, C code for testing implementations of these interfaces and JavaScript that could talk to such implementations from the browser. For reasons of loose coupling it desirable that these three generators could evolve independently. A naive solution might be to copy the in each generator and then reduce it. Since mapping rules are usually written for individual concepts the rule would need to ensure that they only handle nodes that are related to their output. This pollutes the conditions of reduction rule quite quickly and also prevent reuse of generators because they become tightly coupled. It also doesn't solve the problem that the output will all end up in the same `source_gen` folder.

## Generator Configuration Pattern

A pattern that is proven in use in many projects is to separate generation and *the model* from each other. *The model* contains the actual content that we want to generate and then there are (several) models that only contain a concept specifically introduced to configure the generation. 

> *Note: For instance the mbeddrs `BuildConfiguration` is such a concept. While it was not only introduced for this reason it also fulfils this purpose. It can be used to generate tests into a different location than the production code.*

These generations configuration concepts usually contain either a reference to complete model they are supposed to generator or to content of the model that should get generated. This mostly depends on the needs of the users.

The first things that happens during generation is that all the relevant content from *the model* is copied into the currently generated model (the model that only contains the configuration). After that everything in the generator chain works *as usual*. And all generators can safely assume that all the content they are handling is supposed to be handled by these rules. There is no need for additional checking, etc.

This also allows to generate different outputs concurrently. As we have a single model per output we want to produce MPS can generate them concurrently, they only require read access to the model with the actual content. 

There are different ways such generator configurations could refer to the input to generate:

1. A reference to a complete model. `ModelRefExpression` is a handy concept here. 
2. A reference to one or more root nodes of the model get generated. If the language supports cross root node references there are two options as well:
	- References to the roots nodes the user cares about and the generator will copy the dependencies in regardless if the user specified them. 
	- References to all root nodes the generator needs for generation need to be specified explicitly. This is often desired in case the generator configuration has more semantics to it then just copying the content. See mbeddr `BuildConfiguration` as an example where the order of these elements for instance is the order in which the C compiler will evaluate the files.

## Implementation

When implementing such generators there are some things to keep in mind. Instead of using the `BuildConfiguration` of mbeddr as an example we are using a very simple excerpt from other generators here:

```
nlist<IConfigItem> configs = model.nodes(IConfigItem); 
 
sequence<node<ConfigRootRef>> refs = configs.rootRefs; 
list<node<Root>> roots = refs.compChunk.toList; 

refs.forEach({~ref => roots.addAll(ref.collectMissingRoots(roots).toList); }); 
 
nlist<> nodes2copy = new nlist<>; 
nodes2copy.addAll(roots); 

nlist<> copies = genContext.copy with trace nodes2copy; 
 
model.roots(<all>).forEach({~it => it.detach; }); 
 
copies.forEach({~it => model.add root(it); });
```

While there is some code to collect the dependencies at first the most important line is: `nlist<> copies = genContext.copy with trace nodes2copy;`. This uses the `copy with trace` method of the generation context rather then iterating over the nodes one by one and call `.copy` on them. This has one major advantage: MPS will take care of changing the references in the nodes that are copied. This way all references to nodes that are in the list of nodes to be copied are then pointing to the copy. This saves a lot of effort in setting references manually after copying the nodes manually. Because we want the references in the model that we generate to point in that model and not to nodes from the original input. Otherwise we wouldn't be able to use mapping labels to look out the output that ware produced for node during generation.