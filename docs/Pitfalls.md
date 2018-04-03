# Common Pitfalls
The Generator language has a couple of pecularities that inexperienced developers tend to waste a lot of time on.  This listing is by no means comprehensive, but is provided in the hope that it might save future users of the Generator language some nerves.

## Broken References
In MPS, References break when their target node is detached. This does not only happen when explicitly calling the .detach()-method on a node, but also happens automatically when 

1. an existing node is assigned to a position in another Model. When moving nodes between models it is hence strongly recommended to copy them using the .copy-method
1. the target node is contained in a root that is deleted (for instance via an Abandon-Rule or because of a Root-Mapping-Rule with “keep input root=false”). It is hence recommended to use “keep input root=true” whenever possible and to use Abandon Rules with caution.
1. The target node is replaced using the .replace_with() or .replace_with_new() methods.
1. A Reduction Rule is applied to the target node.

## Tracing
When running a generator, an input model is transformed step-wise into an output model and in the process copied at least once per step. For debugging purposes, MPS maintains tracing information along this process, that connects each node with its predecessors / successors in previous / subsequent steps.
This information can also be used within the generator itself, to trace nodes backwards to their origin using genContext.get original copied input by output() or forwards to the output model of the current step using genContext.get copied output by input().

MPS is able to maintain this tracing information fully automatic as long as no baseLanguage is involved. However, when pre-/postprocessing-Skripts or baseLanguage-Code within Makros modifies the model, the chain of tracing information may be broken and affected nodes can no longer be traced (the methods mentioned above will simply return the node they were given as an argument in that case).

There is, however, one methods for maintaining trace information even when modifying the model with baseLanguage: genContext.copy with trace(node) will create a copy of node that contrary to node.copy allows the resulting node to be traced. It is hence recommended to use copy with trace instead of copy whenever possible.

## Mapping Labels 

### Order of Evaluation 
In general, the order of evaluation for Generator Templates is undefined. There is no reliable rule like “left-to-right”, “top-to-bottom”, etc. Even though the order seems to be deterministic within an MPS instance on one particular computer it can vary between computers and versions of MPS. Most probably this is an intentional design decision that should make MPS Templates highly parallelizable. However, updating a mapping label constitutes a side-effect and side-effects somewhat collide with this concept of parallelism. It is hence EXTREMELY IMPORTANT to use other mechanisms for ensuring that mapping labels are filled prior to being accessed.


Two Mechanisms that work well in this respect are


- $MAP_SRC$-Makros have post-processing-method that is invoked in the very last phase of generation. A good strategy for reading values from a mapping label is hence to
	1. Use a $MAP_SRC$-Makro to create concepts that must be retrieved from a mapping label.
	1. In the mapping-method, create an instance of an intermediate concept (often called a “Proxy”) to store all information necessary to retrieve the actual result from the mapping label.
	1. In the post-processing-method, retrieve the actual result from the mapping label (which in this phase has surely been filled) and substitute it for the outputNode (which is the Proxy) using the .replace with-method.
- Reference Macros: When no additional information is necessary to retrieve a result from a mapping label, one can also use a Reference Makro. Reference Makros are also resolved in the very last phase of generation and hence also provide a safe way to use mapping labels.

### Implicit node copies

When running a generator, MPS arranges the mapping configurations into so-called “steps”. Each step takes an input model and transforms it into an output model, which is then used as the input model for the next subsequent step. Preprocessing / postprocessing skripts are run in their own mini-step before / after the mapping configuration. Since it is possible for a generator to also modify its input-model, MPS protects the generation input (lets call it M) by copying it into a first transient model (called M@0) as a very first step. For this reason, no template (not even the very first one) applied within a generator does work on the “original” model. They all work on copies thereof.

It follows that, when using a mapping label to map some sort of source nodes to some sort of target nodes, then these source nodes do NOT come from the original input model, but from some intermediate transient model (by the way: One can determine the model any node in MPS “lives in”, using the node.model-syntax). 

<!-- Furthermore, since mapping labels are implemented using Java HashMaps, querying the mapping label with any other node (even when it is a copy of the one used as a key and hence has the same concept, NodeID, etc.) will NOT work. -->

Fortunately, when trace information is maintained properly (see Section “Tracing” above), then Method genContext.get original copied input by output for (node) provides a way to trace every node in our intermediate model back to a node in the original input model that it was derived from. A good strategy to avoid the problem outlined above where the constant copying of nodes practically prohibits the use of mapping labels, is to

1. properly maintain tracing information.
2. use genContext.get original copied input by output for (node) on the nodes used as keys in the mapping label. For instance, when providing a $LOOP$-Makro with a mapping label, then the “iteration sequence”-method should apply it to every element of its result-sequence as those elements will be used as keys in the mapping label.
3. also use genContext.get original copied input by output for (node) to the node used as key when querying the mapping label. This will ensure that both the key stored in the HashMap as well as the node used for querying are from the same model and - since NodeIDs are unique within a model, having the same nodeID implies that they are the same object.
4. Since the value retrieved from a mapping label might also be nodes from some intermediate transient model, it is often necessary to apply genContext.get copied output by input for (node) to the result to ensure that references do not cross model boundaries.

### Template Calls with Variables
From time to time it becomes necessary to factor some part of a larger template into a separate template fragment to make it reusable. Such template fragments can be invoked using the $CALL$ Node Macro and can also take parameters, just like functions. However, for some reason passing the value of a variable (defines using the $VAR$ Node Macro) as an actual argument causes an IllegalArgumentException, which in turn causes the result of this template call to be discarded.
As an Example:

```
$VAR$ x $CALL$ template

Inspector of $VAR$
     | type: <result-type>
     | value: (genContext, operationContext, node) -> <result-type> { <expression> }

Inspector of $CALL$
     | template: template(genContext.x)
```
will cause an IllegalArgumentException, while
```
$CALL$ template

Inspector of $CALL$
     | template: template(<expression>)
```

works, although the two should be equivalent.
Unfortunately, I cannot tell why this happens, I just encountered this phenomenon several times and was always able to get things to run by removing the $VAR$ and replacing all its occurrences with the <expression> used to initialize it.
