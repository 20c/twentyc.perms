# Install

```sh
pip install grainy
```

# Usage example

```py
from grainy.core import PermissionSet
from grainy.const import *

pset = PermissionSet(
  {
"a" : PERM_READ,
"a.b" : PERM_RW,
"a.b.c" : PERM_DENY,
"b" : PERM_READ,
"b.*.a" : PERM_RW
  }
)

pset.check("a", PERM_READ) # True
pset.check("a.b", PERM_READ) # True
pset.check("a.b", PERM_WRITE) # True
pset.check("a.b.c", PERM_READ) # False
pset.check("a.b.c", PERM_WRITE) # False
pset.check("a.x", PERM_READ) # True
pset.check("b.a.a", PERM_RW) # True
pset.check("b.b.a", PERM_RW) # True
pset.check("b.b.b", PERM_RW) # False

pset.check("a", PERM_READ, explicit=True) # True
pset.check("a.b", PERM_READ, explicit=True) # True
pset.check("a.c", PERM_READ, explicit=True) # False

pset.check("a.?.c", PERM_READ) # False
pset.check("b.?.a", PERM_RW) # True
pset.check("a.?", PERM_RW) # True
```

# Setting and Deleting

```py
pset["a.b"] = const.PERM_READ

del pset["a.b"]

pset.update(
  {
    "a.b" : const.PERM_READ
  }
)

assert "a.b" in pset
```

# Applying to data

You can apply the permissions stored in the permission set to any data dict and data that the permission set does not have READ access to will be removed.

grainy was created out of a need to apply granular permissions on potentially large dict objects and perform well.

```py
# init
pset = core.PermissionSet(
  {
    "a" : const.PERM_READ,
    "a.b.c" : const.PERM_RW,
    "a.b.e" : const.PERM_DENY,
    "a.b.*.d" : const.PERM_DENY,
    "f.g" : const.PERM_READ
  }
)

# original data
data = {
  "a" : {
    "b" : {
      "c" : {
        "A" : True
      },
      "d" : {
        "A" : True
      },
      "e" : {
        "A" : False
      }
    }
  },
  "f": {
    "a" : False,
    "g" : True
  }
}

# expected data after permissions are appied
expected = {
  "a" : {
    "b" : {
      "c" : {
        "A" : True
      },
      "d" : {
        "A" : True
      }
    }
  },
  "f": {
    "g" : True
  }
}

rv = pset.apply(data)
assert rv == expected
```

As of version 1.2 it is also possible to apply permissions to lists using namespace handlers

```py
pset = core.PermissionSet({
    "a.b" : const.PERM_READ,
    "x" : const.PERM_READ,
    "x.z" : const.PERM_DENY,
    "nested.*.data.public" : const.PERM_READ
})

data= {
    "a" : [
        { "id" : "b" },
        { "id" : "c" }
    ],
    "x" : [
        { "custom" : "y" },
        { "custom" : "z" }
    ],
    "nested" : [
        {
            "data" : [
                {
                    "level" : "public",
                    "some" : "data"
                },
                {
                    "level" : "private",
                    "sekret" : "data"
                }
            ]
        }
    ]
}

expected = {
    "a" : [
        { "id" : "b" }
    ],
    "nested" : [
        {
            "data" : [
                { "level" : "public", "some" : "data" }
            ]
        }
    ],
    "x" : [
        { "custom" : "y" }
    ]
}

applicator = core.Applicator()
applicator.handle("x", key=lambda row,idx: row["custom"])
applicator.handle("nested.*.data", key=lambda row,idx: row["level"])

rv = pset.apply(data, applicator=applicator)
assert rv == expected
```


