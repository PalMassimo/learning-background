# OPA Background

## Hello World

A first rego policy

```rego
package policy # like namespace

# the rule will return true if all statements inside are valuated as true
allow = true {
    1 == 1
    3 == 3
}
```

A rule can return any value, not just a boolean

```rego
package policy

allow = "hello" {
    1 == 1
    3 == 3
}
```

To evaluate rules run

```bash
$ opa eval --data helloworld.rego 'data.policy.allow'  # to evaluate rule named 'allow'
```

With reference to the last command, if true the output  will be

```json
{
  "result": [
    {
      "expressions": [
        {
          "value": "hello",
          "text": "data.policy.allow2",
          "location": {
            "row": 1,
            "col": 1
          }
        }
      ]
    }
  ]
}
```

otherwise, the output will be just an empty object `{}`

To avoid an empty response, we can add a default value

```rego
package policy

default allow = false

allow {
    1 == 1
    3 == 3
}
```

To get only the result of the action rule

```bash
$ opa eval --format raw --data policy.rego 'data.policy.allow'
```

## Simple Policy with input data
Let's write a policy that receive a json input and that return true if the input has a field with an array named `rules` that in turn has an element `user`, no matter the position in the array 

```rego
package policy

default allow = false

allow {
    input.rules[_] = "user"
}
```

Let's write down the input in a json file `input.json`

```json
{
    "username": "Massimo",
    "rules": ["user", "officer"]
}
```

To run the evaluation, we have to specify the input

```bash
$ opa eval --format raw --data policy.rego --input input.json 'data.policy.allow'
```

## And and Or conditions
Let's assume that we want that a rule is true if one of two criteria are met. In this case, we can define two rules with the same name.

If we want to have an `and` condition just add statement inside the rule as we have done until now.

# Global Variables
One of the global variables in OPA is the `data` variable, that we often have to prefix (e.g. `data.package_name.rule_name`).

The `data` variable contains all policies and all data that we provide to the running OPA instance.

Another global variable is the `input` variable.

# Testing
To test policies we basically can create another policy file. Let's assume we want to test the following policy

```rego
package policy

default allow = false

allow {
    input.username = "massimo"
}
```


Rego test rule starts with `test_`


```rego
package policy_test

import data.policy.allow

# notice that no input provided
test_allow_is_false_by_default {
    allow == false
    # not allow    # equivalent
}

test_allow_if_user_is_massimo {
    # we can just write 'allow' without '== true'
    allow == true with input as {
        "username": "massimo",
        "roles": ["admin"]
    }
}

test_allow_if_user_is_not_massimo {
    # we can just write 'allow' without '== true'
    not allow with input as {
        "username": "bianca",
        "roles": []
    }
}

test_allow_if_rules_are_ok {
    allow with input as {
        "username": "massimo",
        "rules": ["admin", "officer"]
    }
    not allow with input as {
        "username": "massimo",
        "rules": ["officer"]
    }
}
```

To run the tests we can just run the command

```bash
$ opa test .
PASS 3/3
```

# OPA Server
To run OPA as a server we just run

```bash
$ opa run --server .
```

We can query to the server requests for policy evaluation. Let's assume the server has the following policy

```rego
package policy

default allow := false

allow {
	input.username == "massimo"
    input.roles[_] == "admin"
}
```

To test this we can open Postman and run

```rest
PUT http://localhost:8181/v1/data/policy/allow
{
    "input": {
        "username": "massimo",
        "roles": ["admin"]
    }
}
```

and we will get a `true` value.