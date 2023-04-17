---
layout: single
title: Testing Infrastructure as Code with Python
toc: true
tags: terraform python testing
---

Infrastructure code is considered 100% code but often good software principle, like **testing**, are not applied. Apart from checking the correctness of code, testing documents how code should behave, as well.
In this post I will show how to test a Terraform module using the Python module **tftest** along with the testing framework **pytest**.

# Terraform module

A Terraform module is just a folder containing some Terraform configuration files.
The source code of the module is stored on [Github](https://github.com/andregri/terraform-aws-ec2-dr).
This module creates two AWS instances in two different zones *a* and *b* of the region selected by the user.

```hcl
# omitted...

locals {
  zones = {
    main      = "${var.region}a"
    secondary = "${var.region}b"
  }
}

resource "aws_instance" "ubuntu" {
  for_each = local.zones

  ami                         = data.aws_ami.ubuntu.id
  associate_public_ip_address = true
  availability_zone           = each.value
  instance_type               = var.instance_type

  tags = {
    Name = "instance-${each.key}"
  }
}
```

The Terraform files that use the module are located under `tests/fixtures/plan`.
The `main.tf` creates the two instances:
```hcl
terraform {
  required_version = ">= 1.0"
}

module "test" {
  source        = "../../.."
  instance_type = var.instance_type
  region        = var.region
}
```

# Test fixtures

The scope of tests is to check the system output based on input **stimuli**.

Each test is made of three basic steps:
- **arrange**: prepare input data for the function
- **act**: run the function under test
- **assert**: verify that the test output is what we expected.

Data to be passed to tests are arranged using **fixtures**.

**Fixtures are functions, which will run before each test function to which it is applied.** Fixtures are used to **feed some data to the tests such as database connections, URLs** to test and some sort of input data. Therefore, **instead of running the same code for every test**, we can attach fixture functions to the tests and it will run and return the data to the test **before executing each test**.

In pytest, we can define the fixture functions in `tests/conftest.py` to make them accessible across multiple test files.
The following fixture returns the path of the directory containing the Terraform configuration files:
```python
@pytest.fixture(scope='session')
def fixtures_dir():
  return os.path.join(os.path.dirname(os.path.abspath(__file__)), 'fixtures')
```

To generate the Terraform plan for tests, we use another fixture at `tests/test_plan.py` that calls the TerraformTest function and sets the tfvar file `plan.auto.tfvars`:
```python
@pytest.fixture
def plan(fixtures_dir):
  tf = tftest.TerraformTest('plan', fixtures_dir)
  tf.setup(extra_files=['plan.auto.tfvars'])
  return tf.plan(output=True)
```

# tftest

## Test the Terraform plan

The python module tftest is heavily inspired by two projects: Terratest for the lightweight approach to testing Terraform, and python-terraform for wrapping the Terraform command in Python.

For accessing variables from Terraform plan:
```python
plan.variables['instance_type']
```
In my module, to test the variable values during plan:
```python
def test_variables(plan):
  assert 'instance_type' in plan.variables
  assert plan.variables['instance_type'] == 't2.micro'

  assert 'region' in plan.variables
  assert plan.variables['region'] == 'us-east-1'
```

For accessing outputs from Terraform plan:
```python
plan.outputs['main_arn']
```
Since the outputs are not known during plan, we can just check that outputs are defined in plan:
```python
def test_outputs(plan):
  assert 'main_arn' in plan.outputs
  assert 'main_ip_address' in plan.outputs
  assert 'secondary_arn' in plan.outputs
  assert 'secondary_ip_address' in plan.outputs
```

For accessing resources, e.g. `"google_project_iam_member.test_root_resource"`:
```python
plan.resources['google_project_iam_member.test_root_resource']
```

For accessing modules like `"module.test"`:
```python
plan.modules['module.test']
```

And for accessing module resources:
```python
plan.modules['module.test'].resources
```
Below we check if the module is creating two `"aws_instance"` resources:
```python
def test_modules(plan):
  mod = plan.modules['module.test']
  print(set(mod.resources.keys()))
  assert set(mod.resources.keys()) == set(
      ['aws_instance.ubuntu["main"]',
      'aws_instance.ubuntu["secondary"]'])
```

# Conclusion

In this post I explained how to use the tftest python module to test the plan output of a terraform module.