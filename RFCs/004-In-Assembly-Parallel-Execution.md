# RFC 004 - In-assembly Parallel Execution

## Motivation
The key motivation is to complete the execution of a suite of tests, within a single container, faster.

Coarse-grained parallelization is already supported by vstest, and is available to all test frameworks. That works by launching test execution on each available core as a distinct process, and handing it a container worth of tests (assembly, DLL, or relevant artifact containing the tests to execute) to execute. The unit of isolation is a process. The unit of scheduling is a test container. You can read more about that in our [blogpost](https://blogs.msdn.microsoft.com/visualstudioalm/2016/10/10/parallel-test-execution/).

This document is about providing __finer-grained control__ over parallel execution __via in-assembly parallel execution of tests__ – it enables running tests within an assembly in parallel.

## Requirements:
1. **Easy onboarding** - it should be possible to enable parallel execution for existing MSTest V2 code. For e.g. there might be 10s of 100s of test projects participating in a test run - insisting that all of them make changes to their source code to enable parallelism is a barrier to onbarding the feature.
2. **Fine grained control** - there might still be certain assemblies, or test classes or test methods within the assembly, that might not be ready for execution in parallel. It should be possible for such artifacts to opt-out of parallel execution. Conversly, there might be only a few assemblies that want to opt-in to parallel execution - that should also be possible.
3. **Override** - Parallel execution will have an impact on data collectors. Since test execution will be in parallel, the start/end events marking the execution of a particular test might get interleaved with those of any other test that might be executing in parallel. Therefore it shoud be possible for a feature that requires data collection to override and OFF all parallel execution. An example of a feature that might want to do this would be TIA (Test Impact Analysis).
4. **Test lifecycle semantics** - we will need to clarify the semantics to the various xxxInitialize/xxxCleanup methods.

## Approach
The simplest way to enable in-assembly parallel execution is to enable it globally for all MSTest V2 test assemblies using a .runsettings file as follows:
```xml
<RunSettings>
<!-- MSTest adapter -->  
  <MSTest>
    <Parallelize>
      <Workers>4</Workers>
      <Scope>ClassLevel</Scope>
    </Parallelize>
  </MSTest>
</RunSettings>
```
From the CLI these values can be provided using the "--" syntax.

This is as if every assembly were annotated with the following:
```csharp
[assembly: Parallelize(Workers = 4, Scope = ExecutionScope.ClassLevel)]
```

Parallel execution will be realized by spawning the appropriate number of worker threads (4), and handing them tests at the specified scope.

There will be 3 scopes of parallelization supported:
- ClassLevel - each thread of execution will be handed a TestClass worth of tests to execute. Within the TestClass, the test methods will execute serially. This will be the default - tests witin a class might have interdependency, and we don't want to be too aggressive.
- MethodLevel - each thread of execution will be handed TestMethods to execute.
- Custom - the user will provide plugins implementing the required execution semantics. This will be covered in a separate RFC. 

The value for the number of worker threads to spawn to execute tests can be set using a single assembly level attribute that will take a parameter whose values can be as follows:
- 0 - Auto configure; use as many tests as possible based on CPU and core count.
- n - The number n of threads to spawn to executes tests.

An assembly/Class/Method can explicitly opt-out of parallelization using an attribute that will indicate that it may not be run in parallel with any other tests. The attribute does not take any arguments, and may be added at the method, class, or assembly level.
```csharp
[DoNotParallelize()]
```
When used at the assembly level, all tests within the assembly will be executed serially.
When used at the Class level, all tests within the class will be executed serially after the parallel execution of all other tests is completed.
When used at the Method level, the test method will be executed serially after the parallel execution of all other tests is completed.

This can be specified via a .runsettings as a global override to turn OFF parallelization as follows:
```xml
<RunSettings>  
  <!-- Configurations that affect the Test Framework -->  
  <RunConfiguration>  
    <DoNotParallelize/> 
  </RunConfiguration>
</RunSettings>
```

Test lifecyle method semantics
- AssemblyInitialize/Cleanup shall be run only once per assembly (irrespective of parallel or not).
- ClassInitialize/Cleanup shall be run only once per class (irrespective of parallel or not).
- TestInitialize/Cleanup shall be run only once per method.


## Notes
1. It will up to the user to ensure that the tests are parallel-ready before enabling parallel test execution.
2. Features that rely on data collectors will need to globally turn OFF parallel execution. They can do so by either crafting a .runsettings file as shown above, or by passing the the "--"syntax from the CLI. For e.g. the VSTest task with Test Impact Analysis ON will need to do this when invoking vstest runner.
3. Diagnosing test failures during parallel execution will require appropriately formatted logging. The adapter should take care to straighten out the logs and emit them appropriately formatted.
