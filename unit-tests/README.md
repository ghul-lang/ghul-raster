# Unit tests

This folder holds unit tests, which are driven by the MSTest framework.

Writing unit tests in ghūl is currently a bit awkward. This is because ghūl doesn't yet have support for expression trees or for static calls to generic methods with explicit type arguments. Unfortunately Moq relies heavily on expression trees and mocking, assertion and IoC frameworks in general tend to have interfaces requiring static calls to generic methods where the type arguments cannot be inferred and type constraints are not enforced.

Nevertheless, unit tests are possible - there are more detailed examples in the unit tests for the [ghul-test project](https://github.com/ghul-lang/ghul-test/tree/main/tests/src)  

