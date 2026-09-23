<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: 2026 The Linux Foundation
-->

# test-python-submodules

A fixture for testing that scan lanes check out **submodules**,
including nested ones, before they scan. Used by
[`lfreleng-actions/security-workflows`](https://github.com/lfreleng-actions/security-workflows)
([#74](https://github.com/lfreleng-actions/security-workflows/issues/74)).

## Layout

The fixture nests three levels, and lives in **one repository on one
branch**. Each submodule pins an earlier commit of this same branch:

```text
main (parent)              six==1.17.0
└── direct/   → commit 2   iniconfig==2.0.0
    └── nested/ → commit 1 tomli==2.2.1
```

Each level carries a distinct `requirements.txt` pinned with `==`, which
Nexus IQ reads without a build, and a marker source file
(`*_level.py`), which Sonar indexes. So each checkout mode leaves a
different, observable result:

| `submodules` | Content present             |
| ------------ | --------------------------- |
| `false`      | `six`                       |
| `true`       | `six`, `iniconfig`          |
| `recursive`  | `six`, `iniconfig`, `tomli` |

A lane that downgrades `recursive` to `true` without saying so loses
`tomli` and nothing else, and a lane that drops `submodules`
altogether loses both.

## Why one repository

A nested submodule needs a middle level that is itself a
superproject. Pinning ancestors of `main` provides all three levels
without extra repositories or orphan branches, both of which a fork
pull request cannot introduce. The ancestors stay reachable from
`main`, so GitHub serves them by SHA to a shallow submodule fetch.

## Rules for changing this repository

- **Never rewrite or force-push `main`'s first three commits.** The
  submodule pins name them by SHA. Rewriting them orphans the pins and
  every submodule checkout fails.
- **Do not change the pinned versions** without updating the
  assertions in `security-workflows` that expect them.
- Adding files on top of the parent level is fine. Changing a lower
  level means building a new chain of commits, because each level pins
  the exact commit below it.
