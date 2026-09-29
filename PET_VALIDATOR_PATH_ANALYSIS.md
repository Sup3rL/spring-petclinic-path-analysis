# `PetValidator.validate` path analysis

This document applies statement, branch, and loop white-box analysis to
`org.springframework.samples.petclinic.owner.PetValidator.validate(Object, Errors)`.
The method is implemented in
`src/main/java/org/springframework/samples/petclinic/owner/PetValidator.java`;
the existing tests are in
`src/test/java/org/springframework/samples/petclinic/owner/PetValidatorTests.java`.
The analysis treats the `&&` expression as short-circuit control flow, as
required for basis-path analysis.

## Method under analysis

```java
public void validate(Object obj, Errors errors) {
    Pet pet = (Pet) obj;
    String name = pet.getName();

    if (!StringUtils.hasText(name)) {
        errors.rejectValue("name", REQUIRED, REQUIRED);
    }
    else if (name.length() > MAX_NAME_LENGTH) {
        errors.rejectValue("name", "size", "Name must be no more than 30 characters");
    }

    if (pet.isNew() && pet.getType() == null) {
        errors.rejectValue("type", REQUIRED, REQUIRED);
    }

    if (pet.getBirthDate() == null) {
        errors.rejectValue("birthDate", REQUIRED, REQUIRED);
    }
}
```

`Pet.isNew()` is true when its inherited `id` is `null`. The validator always
evaluates the birth-date decision, regardless of the outcomes of the earlier
name and type checks.

## Control-flow graph

### Nodes

| Node | Statement or predicate |
| --- | --- |
| 1 | Entry; cast `obj` to `Pet` and read `name` |
| 2 | `!StringUtils.hasText(name)` is true |
| 3 | Reject the `name` as required |
| 4 | `!StringUtils.hasText(name)` is false; evaluate `name.length() > 30` |
| 5 | Reject the `name` for exceeding the maximum length |
| 6 | Name validation completes without a length rejection |
| 7 | Evaluate `pet.isNew()` |
| 8 | `pet.isNew()` is false (short-circuit skips `pet.getType() == null`) |
| 9 | `pet.isNew()` is true; evaluate `pet.getType() == null` |
| 10 | `pet.getType() == null` is true |
| 11 | Reject `type` as required |
| 12 | `pet.getType() == null` is false |
| 13 | Evaluate `pet.getBirthDate() == null` |
| 14 | Reject `birthDate` as required |
| 15 | Exit |

Nodes 4 and 6 represent the false outcome of the nested name decision:
node 4 is the decision and node 6 is the merge after either the length
decision or its rejection. Similarly, node 9 is reachable only when the
left operand of `&&` is true.

### Edges

The directed edges are:

```text
1→2, 2-T→3, 2-F→4, 3→7,
4-T→5, 4-F→6, 5→6, 6→7,
7-F→8, 7-T→9, 8→13,
9-T→10, 9-F→12, 10→11, 11→13, 12→13,
13-T→14, 13-F→15, 14→15
```

The graph deliberately has no edge from node 8 to node 10: when
`pet.isNew()` is false, Java short-circuits `pet.getType() == null` and the
right operand is not evaluated.

### Mermaid CFG

The following diagram uses the same node numbers as the table above. `Yes` and
`No` labels are predicate outcomes; the edge from node 7 to node 8 is the
short-circuit path that avoids evaluating node 9.

```mermaid
flowchart TD
    N1([1 Entry: cast obj; read name]) --> N2{2 P1: !hasText(name)?}
    N2 -- Yes --> N3[3 Reject name required]
    N2 -- No --> N4{4 P2: name.length() > 30?}
    N3 --> N7{7 P3: pet.isNew()?}
    N4 -- Yes --> N5[5 Reject name size]
    N4 -- No --> N6[6 Name validation complete]
    N5 --> N6
    N6 --> N7
    N7 -- No / short-circuit --> N8[8 Skip P4]
    N7 -- Yes --> N9{9 P4: pet.getType() == null?}
    N8 --> N13{13 P5: birthDate == null?}
    N9 -- Yes --> N10[10 Missing type]
    N9 -- No --> N12[12 Type present]
    N10 --> N11[11 Reject type required]
    N11 --> N13
    N12 --> N13
    N13 -- Yes --> N14[14 Reject birthDate required]
    N13 -- No --> N15([15 Exit])
    N14 --> N15
```

### TriClass-style numbered layout

The same CFG can be read top-to-bottom as a numbered layout. This makes the
branch and short-circuit behavior explicit even in a renderer that does not
display Mermaid diagrams:

```text
 [1] Entry
   |
   v
 [2] P1: !hasText(name)?
   | Yes                         | No
   v                             v
 [3] Reject name required      [4] P2: name.length() > 30?
   |                             | Yes              | No
   |                             v                  v
   |                          [5] Reject size     [6] Name complete
   |                             |                  |
   +-----------------------------+------------------+
                                 v
                         [7] P3: pet.isNew()?
                          | No (short-circuit)     | Yes
                          v                        v
                    [8] Skip P4             [9] P4: type == null?
                          |                  | Yes             | No
                          |                  v                 v
                          |             [10] Missing type  [12] Type present
                          |                  |                 |
                          |                  v                 |
                          |             [11] Reject type       |
                          |                  |                 |
                          +------------------+-----------------+
                                             v
                                  [13] P5: birthDate == null?
                                    | Yes                 | No
                                    v                     v
                         [14] Reject birthDate       [15] Exit
                                    |
                                    v
                               [15] Exit
```

In particular, `7 -- No` means that `pet.getType() == null` is not evaluated;
it is not a false outcome for P4. A P4 outcome exists only on the `7 -- Yes`
edge.

## Predicate outcomes and coverage

| Predicate | True outcome | False outcome | Observable effect |
| --- | --- | --- | --- |
| P1: `!StringUtils.hasText(name)` | `null`, empty, or whitespace-only name; reject `name` as required | non-blank name; evaluate P2 | Selects the required-name branch versus the `else if` |
| P2: `name.length() > 30` | reject `name` with the size error | no name error | Only evaluated when P1 is false |
| P3: `pet.isNew()` (left operand of `&&`) | evaluate P4 | skip P4 and type rejection | Existing pets do not require a type in this method |
| P4: `pet.getType() == null` (right operand of `&&`) | reject `type` as required | no type error | Only evaluated when P3 is true |
| P5: `pet.getBirthDate() == null` | reject `birthDate` as required | no birth-date error | Always evaluated |

For branch coverage, each listed true and false outcome must occur at least
once. For statement coverage, each reject statement and the normal continuation
edges must execute at least once. The currently checked-in tests cover the
blank-name, long-name, new-pet/missing-type, missing-birth-date, and all-valid
outcomes, but do not explicitly cover a non-new pet (P3 false), a new pet with
a non-null type (P4 false), or a whitespace-only name.

## Cyclomatic complexity

Using the predicate-node form of the CFG:

```text
V(G) = E - N + 2
     = 19 - 15 + 2
     = 6
```

The edge count includes the merge nodes but does not count them as additional
decisions. The equivalent conventional McCabe calculation counts the five
independent decisions (P1, P2, P3, P4, and P5):

```text
V(G) = number of decisions + 1
     = 5 + 1
     = 6
```

Nodes 4, 6, 9, and 12 model expression evaluation and merge detail; they do
not add source-level decisions.

## Basis set of linearly independent paths

`E` means the corresponding error is recorded; `N` means no error is recorded.
The path labels list predicate outcomes in evaluation order. Each path is
feasible, and together they exercise every decision outcome and the
short-circuit edge.

| Path | Predicate outcomes | Execution outline | Concrete input conditions |
| --- | --- | --- | --- |
| B1 | P1=T, P3=F, P5=F | required-name rejection → existing pet skips P4 → exit | `name=""`, `id=1`, `birthDate=2020-01-01`; expected `name=E`, no `type` or `birthDate` error |
| B2 | P1=T, P3=T, P4=T, P5=T | required-name rejection → missing type rejection → missing birth-date rejection → exit | `name=" "`, `id=null`, `type=null`, `birthDate=null`; expected all three field errors |
| B3 | P1=F, P2=F, P3=F, P5=F | valid short name → existing pet skips P4 → exit | `name="Buddy"`, `id=1`, `birthDate=2020-01-01`; expected no errors |
| B4 | P1=F, P2=T, P3=T, P4=F, P5=F | long-name rejection → new pet with type → exit | `name` is 31 characters, `id=null`, non-null `type`, non-null `birthDate`; expected only `name=size` |
| B5 | P1=F, P2=F, P3=T, P4=T, P5=F | valid short name → missing type rejection → exit | `name="Buddy"`, `id=null`, `type=null`, `birthDate=2020-01-01`; expected only `type=required` |
| B6 | P1=F, P2=F, P3=T, P4=F, P5=T | valid short name → new pet with type → missing birth-date rejection → exit | `name="Buddy"`, `id=null`, non-null `type`, `birthDate=null`; expected only `birthDate=required` |

B1 supplies the P3-false short-circuit edge; B4 and B6 supply the P4-false
edge; B2 and B5 supply the P4-true edge. B3 is the all-normal baseline, and
the remaining paths add the independent name and birth-date outcomes.

## Loop coverage

`validate` contains no explicit loop (`for`, enhanced `for`, `while`, or
`do-while`), and none of its called validation expressions is a loop in this
method's control-flow graph. Therefore loop coverage is not applicable:
there are no zero-, one-, or many-iteration cases to construct. The `&&`
operator is a short-circuit conditional, not a loop, and must not be counted
as one.

## Mapping to existing tests

| Existing test | Relevant basis/outcome |
| --- | --- |
| `validate` | B3-like all-valid case (the fixture creates a new pet, so P3 is true and P4 is false) |
| `validateWithInvalidPetName` | B2's P1 true outcome with the other fields valid |
| `validateWithInvalidPetType` | B5's P3 true/P4 true outcome |
| `validateWithInvalidBirthDate` | B6-like P5 true outcome, with a non-null type |
| `validateWithLongPetName` | B4's P2 true outcome |

The tests assert field-error presence rather than every field's complete error
set. The concrete conditions in the basis table specify the expected mapping
needed for full statement and branch coverage.
