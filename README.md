# Good software development practices

[![Run in Cisco Cloud IDE](https://static.production.devnetcloud.com/codeexchange/assets/images/devnet-runable-icon.svg)](https://developer.cisco.com/codeexchange/devenv/NSO-developer/nso-service-dev-practices)

Learn more about software development practices you should follow when developing NSO services.

[Click here](https://developer.cisco.com/codeexchange/devenv/NSO-developer/nso-service-dev-practices) to practice this Lab on the NSO Playground.

## Objectives

After completing this Lab you will:

- Be able to use Pylint for static analysis of your code
- Learn how to refactor your code for better reusability, maintainability and performance
- Be able to test your code better

# Lint your code

Pylint is a widely used static code analysis tool for Python programming. It assists developers in improving the quality and maintainability of their Python code by analyzing it for potential errors, style inconsistencies, and adherence to coding standards. Pylint evaluates code against a set of predefined rules and conventions, identifying issues such as syntax errors, variable naming inconsistencies, unused variables, and more. It generates detailed reports and provides suggestions for code improvements, helping developers write cleaner, more readable, and efficient Python code. Pylint contributes to better code quality, reduced bugs, and enhanced code maintainability across projects.

When writing Python code for service mapping or other NSO development Pylint is a great tool to use. In the following example you will use it to lint your package code as part of package build process.

Usually packages are built as part of a CI job. By adding the pylint to the `all` target of a package you will make sure the Python code is always linted. With that you prevent that syntactic errors or code ends up in a merge request since for a successful pipeline run you have to fix them.
## Use Pylint for NSO Python package analysis

You will learn how to use Pylint on an existing NSO package that uses Python to implement mapping logic. In the `~/src/nso-service-dev-practices` folder you will find the sample loopback package you will work with in this lab.

### Modify package Makefile to include pylint

First add a recipe for running Pylint to the package Makefile. Open Makefile under `~/src/nso-service-dev-practices/loopback/src/Makefile` and add the the call for `pylint` target to the `all:` target on the first line:

```Makefile
all: fxs pylint
```

Now add the `pylint` target to the bottom of the `Makefile`, it should look like the following snippet:

```Makefile
pylint:
	pylint --rcfile=.pylintrc ../python/loopback
```

In the pylint target you have specified the .pylintrc file that should be used for linting this package. Generate the rcfile using pylint:

```bash
pylint --generate-rcfile > ~/src/nso-service-dev-practices/loopback/src/.pylintrc
```

Build the package using `all` target, this will also trigger the `pylint` target you added as a dependency:
```
make -C ~/src/nso-service-dev-practices/loopback/src/ all
```
Output:
```
developer:~ > make -C ~/src/nso-service-dev-practices/loopback/src/ all
make: Entering directory '/home/developer/src/nso-service-dev-practices/loopback/src'
pylint --rcfile=.pylintrc ../python/loopback
************* Module loopback.loopback
/home/developer/src/loopback/python/loopback/loopback.py:1:0: C0114: Missing module docstring (missing-module-docstring)
/home/developer/src/loopback/python/loopback/loopback.py:6:0: C0115: Missing class docstring (missing-class-docstring)
/home/developer/src/loopback/python/loopback/loopback.py:8:4: C0116: Missing function or method docstring (missing-function-docstring)
/home/developer/src/loopback/python/loopback/loopback.py:20:8: W0622: Redefining built-in 'vars' (redefined-builtin)
/home/developer/src/loopback/python/loopback/loopback.py:9:49: W0212: Access to a protected member _path of a client class (protected-access)
/home/developer/src/loopback/python/loopback/loopback.py:29:0: C0115: Missing class docstring (missing-class-docstring)
/home/developer/src/loopback/python/loopback/loopback.py:3:0: C0411: standard import "import ipaddress" should be placed before "import ncs" (wrong-import-order)

------------------------------------------------------------------
Your code has been rated at 6.96/10 (previous run: 6.96/10, +0.00)

make: *** [Makefile:32: pylint] Error 20
make: Leaving directory '/home/developer/src/loopback/src'
```

As you can see pylint detected multiple problems with the loopback package Python code. You will fix some of these problems and change the rcfile to ignore the rest of them.

### Fix problems in Python code

In the printout above you can see that pylint emits different types of messages:

* E: Error - These messages indicate critical issues that are likely to cause problems or errors in your code.
* W: Warning - These messages point out potential issues or code smells that might lead to problems but are not as critical as errors.
* C: Convention - These messages relate to style and coding convention recommendations. They help ensure that your code adheres to a consistent style, making it more readable and maintainable.

You can see that there are no errors detected in the loopback package, so lets first try to fix the warnings.

The first warning message is:
```
W0622: Redefining built-in 'vars' (redefined-builtin)
/home/developer/src/loopback/python/loopback/loopback.py:8:4: C0116:
```

It says that we are using a variable name that is part of Python built-in keywords. You should avoid that to prevent confusion. Pylint also reports filename and the line number where the problem was detected.

Open the file with the problem and rename the problematic variable from `vars` to `tvars`. Your code should look like this:

```python
        bgp_address = list(net.hosts())[0]
        tvars = ncs.template.Variables()
        tvars.add('MANAGEMENT_ADDRESS', management_address)
        tvars.add('BGP_ADDRESS', bgp_address)
        template.apply('loopback-template', tvars)
```

Re-run the build of the package. The warning should go away and the score should be higher now.

The next error is about the access to the protected member of a client class - class attributes that starts with the underscore should not be accessed by the user code.
```
/home/developer/src/loopback/python/loopback/loopback.py:9:49: W0212: Access to a protected member _path of a client class (protected-access)
```

Let's say you want to make an exception here and keep the access to the `_path` attribute anyway. In this case you can instruct the linter to ignore this line for a specific problem by using the following comment syntax at the end of the problematic line:

```python
        self.log.info('Service create(service=', service._path, ')') # pylint: disable=protected-access
```

Re-run the linter and you will see that now only the convention problems remains. Fix the import ordering problem by placing the import of the ipaddress package before the `import ncs`. The convention is to import Python standard libraries before third party imports.

```python
# -*- mode: python; python-indent: 4 -*-
import ipaddress
import ncs
from ncs.application import Service
```
There are now only documentation related convention messages. In case you want to ignore them package wide you can suppress them by changing the `.pylintrc` file. Open it and find keyword `disable` under MESSAGES CONTROL section. Add the convention messages that are still triggered:
```diff
disable=raw-checker-failed,
        bad-inline-option,
        locally-disabled,
        file-ignored,
        suppressed-message,
        useless-suppression,
        deprecated-pragma,
-       use-symbolic-message-instead
+       use-symbolic-message-instead,
+       C0114,
+       C0115,
+       C0116
```

Re-run the `make all` command. Now all errors are fixed or suppressed and the build is successful.

```bash
make -C ~/src/nso-service-dev-practices/loopback/src/ all
```
Output:
```bash
developer:~ > make -C ~/src/nso-service-dev-practices/loopback/src/ all
pylint --rcfile=.pylintrc ../python/loopback

-------------------------------------------------------------------
Your code has been rated at 10.00/10 (previous run: 7.78/10, +2.22)
```

# Write better Python code

In the following section you will refactor Python NSO service callback function. It is intentionally written in a way that does not follow good practices of modularity, reusability and maintainability. With refactoring you will improve the code so that these good practices can be met.

## Loopback Python example

Open the `~/src/nso-service-dev-practices/loopback/python/loopback/loopback.py` example again and study its contents.

```python
# -*- mode: python; python-indent: 4 -*-
import ipaddress
import ncs
from ncs.application import Service


class ServiceCallbacks(Service):
    @Service.create
    def cb_create(self, tctx, root, service, proplist):
        self.log.info('Service create(service=', service._path, ')') # pylint: disable=protected-access

        management_prefix = service.management_prefix
        self.log.debug(f'Value of management-prefix leaf is {management_prefix}')
        net = ipaddress.IPv4Network(management_prefix)
        management_address = list(net.hosts())[0]

        bgp_prefix = service.bgp_prefix
        self.log.debug(f'Value of bgp-prefix leaf is {bgp_prefix}')
        net = ipaddress.IPv4Network(bgp_prefix)
        bgp_address = list(net.hosts())[0]
        tvars = ncs.template.Variables()
        tvars.add('MANAGEMENT_ADDRESS', management_address)
        tvars.add('BGP_ADDRESS', bgp_address)
        template = ncs.template.Template(service)
        template.apply('loopback-template', tvars)
```

First thing that we can improve in the given code is to use a function that will calculate the management IP address and bgp address from the given prefix. With this we avoid repeating the same code for the calculation multiple times and we can also use it in the future.

The function to calculate the address from the prefix should look like this:
```python
def calculate_ip_address(prefix):
    net = ipaddress.IPv4Network(prefix)
    return list(net.hosts())[0]
```
