# Best Practices
Fixed generator plans:
might work in special cases when 
GenPlans are NOT extensible
The problem is , only the generators are involved which are listed in the genplan
A Problem might appear during the generation of tests. If the genplan is involved in the solution/model then only the generators will be executed that are setup in the genplan. Generators for tests will not be triggered.
		

Use the following code to prevent your generator from being applied to NodeTests: 

```
--- is applicable ---
(genContext)->boolean {
  // will enable the usage of this rootnode in nodetests
  !genContext.originalModel.nodes(<all>).any({~it => it.concept.getLanguage().getQualifiedName().startsWith("jetbrains.mps.lang.test"); });
}
```

## Preprocessing
Pre processing is often an easy 

- Preprocessing scripts
	- manipulate the input model in a way it’s easier to use
	- allow easy debugging on the generated code
	- Intermediate languages for generation
	- sometimes it’s easier to create a new intermediate language in the generator language itself and use preproc. scripts to transform parts of the input model to the intermediate language instances afterwards the reduction rules need only to take care about reducing the intermediate language instances, which makes reduction rules smaller
## Reduction Rules