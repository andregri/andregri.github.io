---
layout: single
title: Mocking Python functions
toc: true
tags: python testing
---

This post shows how to mock one or more Python functions for unit testing.

I am using Python 3.11 and its standard library.

# Python subprocess

I was writing a Python script to execute some kubectl command. Since I couldn't access a cluster to test my script, I thought that I could test it using a mock of kubectl. I didn't need to mock the API request and response, I just needed a JSON response for each kubectl invocation.

The first idea that came to my mind was:

- write a bash function to mock kubectl
- export the function with `export -f kubectl`
- write a bunch of test in Bats

Even if Bats tests were very easy and fast to setup, it didn't work: my Python script called always the real kubectl command, not its mocked version.

My Python script was invoking the kubectl tool using the [subprocess](https://docs.python.org/3/library/subprocess.html) library. I think that this behaviour is due to the fact that subprocess module spawns a new process, so it doesn't inherit the exported variables in the shell.

Then, I searched how to mock subprocess module without success. So, I simplified even more and I decided to mock my Python functions: the mocks should return a different JSON output based on inputs, like a switch-case statement.

# Mocking a python function

The `unittest.mock` module contains the `patch` decorator that is used to patch functions, indeed.

Assume you have a **script.py** that calls kubectl to get namespaces:

```python
# script.py
import json
import subprocess

def get_namespaces(label: str = '') -> list:
  process = subprocess.Popen(
    ['kubectl', 'get', 'namespaces', '-l', label, '-o', 'json'],
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
  )
  stdout, stderr = process.communicate()
  if stderr:
    print(stderr)
    return []
  namespaces = json.loads(stdout.decode('utf-8'))

  return [ns['metadata']['name'] for ns in namespaces['items']]

def complex_tool():
  namespaces = get_namespaces()
  ...
```

Then you create **test_script.py** to test get_namespaces:

```python
import script

import unittest
from unittest.mock import patch

def get_namespaces_return_value():
  with open('tests/fixtures/namespaces.json') as namespaces_file:
    json_data = json.load(namespaces_file)
    return [ns['metadata']['name'] for ns in json_data['items']]

@patch('script.get_namespaces')
def test_complex_tool(
    get_namespaces_mock,
  ):
  """
  Test that get_namespaces returns all namespaces
  """
  get_namespaces_mock.return_value = get_namespaces_return_value()

  out = script.complex_tool()
  # assertions...
```

The steps to mock the `get_namespaces` function are:

- define a function like `get_namespaces_side_effect` that mocks the real function `get_namespaces`. In my case, the mock returns the content of a json file.
- Define the test function like `test_complex_tool` but apply the `patch` decorator that takes as argument the function to be patched, in my case `script.get_namespaces`.
- The test function `test_complex_tool` needs an argument, `get_namespaces_mock`, whose value is set with a Mock object by the patch decorator.
- Inside the test function `test_complex_tool`, modify the behaviour of the mock `get_namespaces_mock` as you need: in my case, I set the return value of the mock to be the string returned by `get_namespaces_return_value`

# Mocking many functions

To mock many functions, the order which you pass the mocks to the test function matters. **The first decorator must be the last function argument**:

```python
@patch('script.get_namespaces')
@patch('script.get_pods')
def test_complex_tool(
    get_pods_mock,
    get_namespaces_mock,
  ):
  """
  Test that get_namespaces returns all namespaces
  """
  get_namespaces_mock.return_value = get_namespaces_return_value()
  get_pods_mock.return_value = get_pods_return_value()

  out = script.complex_tool()
  # assertions...
```

# Parametrize the mocked function

Up to now we customised only the return value of the mock function: the mock returns always the same value whenever it is called. But often our functions can take some inputs and the output depends on the inputs.

To implement a mock with such a behaviour, we can set the `Mock.side_effect` property to a function. For instance, assume the `get_pods` function has an argument to filter pods based on the namespace:

```python
# script.py

def get_pods(namespace: str) -> list:
  ...
  return pods
```

In the **test_script.py**, we define a function that returns different values based on the input namespace:

```python
# test_script.py

def get_pods_side_effect(namespace: str) -> list:
  if namespace == 'dev':
    return ['pod1-dev', 'pod2-dev']
  if namespace == 'prod':
    return ['pod1-prod', 'pod2-prod']
  return ['pod1', 'pod2']
```

Then, in the test cases you set the side_effect property with the `get_pods_side_effect` function: 

```python
@patch('script.get_namespaces')
@patch('script.get_pods')
def test_complex_tool(
    get_pods_mock,
    get_namespaces_mock,
  ):
  """
  Test that get_namespaces returns all namespaces
  """
  get_namespaces_mock.return_value = get_namespaces_return_value()
  get_pods_mock.side_effect = get_pods_return_value

  # act ...
  # assert ...
```

# Conclusion

Python **unittest** module provides a powerful `Mock` object, that allows to mock any function, and the `patch` decorator to be applied to test cases.

To not duplicate the patch decorator to every test case function, it is possible to decorate the test case class directly.
