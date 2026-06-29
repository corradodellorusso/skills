---
name: graphql-ast-visitors
description: >
  Step-by-step guide for mastering GraphQL AST Visitors in JavaScript/TypeScript
  with plain graphql-js — covering visitor patterns, schema AST traversal,
  and operation document traversal.
---

# graphql-ast-visitors

A complete reference for writing and reviewing GraphQL AST Visitors using plain `graphql-js`. Covers how to obtain an AST from both a schema and an operation document, all visitor patterns with precise semantics, node kind reference tables, practical examples, and a review checklist.

## When to use

- When writing a new AST visitor (schema or operations)
- When reviewing an existing visitor for correctness or completeness
- When extracting information from a schema or operation document
- When transforming / rewriting AST nodes

## Instructions

### 1. Get an AST to work with

There are two distinct inputs. Know which one you have before writing the visitor.

#### A. Schema AST — from an SDL string or a compiled `GraphQLSchema`

```js
import { parse } from 'graphql/language';
import { printSchema } from 'graphql/utilities';

// From an SDL string directly
const schemaDoc = parse(sdlString);              // → DocumentNode

// From a compiled GraphQLSchema object
const schemaDoc = parse(printSchema(schema));    // → DocumentNode
```

The resulting `DocumentNode`'s `definitions` array contains **type definition nodes** (`ObjectTypeDefinitionNode`, `InterfaceTypeDefinitionNode`, etc.) and extension nodes — **not** operation nodes.

> Do not use `introspectionFromSchema` for AST work — it returns a plain JS object, not an AST node tree.

#### B. Operation AST — from a query/mutation/subscription string

```js
import { parse } from 'graphql/language';

const operationDoc = parse(queryString);         // → DocumentNode
```

The resulting `DocumentNode`'s `definitions` array contains `OperationDefinitionNode`s and `FragmentDefinitionNode`s.

> Both produce a `DocumentNode`; the `kind` of each entry in `definitions[]` tells you which world you are in.

---

### 2. Core `visit()` API

```ts
import { visit, BREAK } from 'graphql/language';

visit(
  root: ASTNode,
  visitor: Visitor<ASTKindToNode>,
  visitorKeys?: Partial<Record<string, readonly string[]>>
): ASTNode
```

- Performs **depth-first** traversal
- Calls `enter` going down, `leave` going up
- **Returns the (possibly modified) root node** — it does NOT mutate in place

**`ASTVisitFn` full signature:**

```ts
(
  node: TNode,
  key: string | number | undefined,                          // property name or array index in parent
  parent: ASTNode | readonly ASTNode[] | undefined,
  path: ReadonlyArray<string | number>,
  ancestors: ReadonlyArray<ASTNode | readonly ASTNode[]>
) => undefined | false | null | typeof BREAK | ASTNode
```

**Return-value semantics:**

| Return value | Effect |
|---|---|
| `undefined` (or nothing) | No change; continue traversal |
| `false` | Skip this node's entire subtree — **enter only** |
| `BREAK` | Halt the entire traversal immediately |
| `null` | Remove this node from the AST |
| any `ASTNode` | Replace this node with the returned node |

> Returning a value from **`leave`** replaces the node after all its children have been visited.
> Returning a value from **`enter`** replaces before children are visited.

---

### 3. Visitor patterns

**Pattern 1 — KindVisitor shorthand (implies `enter`)**

```js
visit(doc, {
  Field(node) { /* called on enter for every FieldNode */ }
});
```

**Pattern 2 — Explicit enter / leave per Kind**

```js
visit(doc, {
  OperationDefinition: {
    enter(node) { /* called going in */ },
    leave(node) { /* called going out */ }
  }
});
```

Use `enter` when you want to skip a subtree (`return false`).
Use `leave` for bottom-up computation where you need child results first.

**Pattern 3 — Generic catch-all**

```js
visit(doc, {
  enter(node) { /* fires for every node, going down */ },
  leave(node) { /* fires for every node, going up */ }
});
```

**Pattern 4 — Reducer (bottom-up value computation)**

```js
// Replace nodes with computed values on leave
const result = visit(doc, {
  Field: {
    leave(node) {
      return { ...node, name: { ...node.name, value: node.name.value.toUpperCase() } };
    }
  }
});
```

**Pattern 5 — `visitInParallel` (multiple independent concerns in one pass)**

```js
import { visitInParallel } from 'graphql/language';

visit(doc, visitInParallel([visitorA, visitorB]));
```

> **Key caveat:** if `visitorA` edits a node, `visitorB` does **not** see the edited version — each visitor receives the original node as it was at traversal entry.

---

### 4. Schema AST — node kinds and key properties

| Kind constant | Node type | Key properties |
|---|---|---|
| `DOCUMENT` | `DocumentNode` | `definitions` |
| `SCHEMA_DEFINITION` | `SchemaDefinitionNode` | `operationTypes`, `directives` |
| `OBJECT_TYPE_DEFINITION` | `ObjectTypeDefinitionNode` | `name`, `fields`, `interfaces`, `directives` |
| `FIELD_DEFINITION` | `FieldDefinitionNode` | `name`, `type`, `arguments`, `directives` |
| `INTERFACE_TYPE_DEFINITION` | `InterfaceTypeDefinitionNode` | `name`, `fields`, `interfaces`, `directives` |
| `UNION_TYPE_DEFINITION` | `UnionTypeDefinitionNode` | `name`, `types`, `directives` |
| `ENUM_TYPE_DEFINITION` | `EnumTypeDefinitionNode` | `name`, `values`, `directives` |
| `ENUM_VALUE_DEFINITION` | `EnumValueDefinitionNode` | `name`, `directives` |
| `INPUT_OBJECT_TYPE_DEFINITION` | `InputObjectTypeDefinitionNode` | `name`, `fields`, `directives` |
| `INPUT_VALUE_DEFINITION` | `InputValueDefinitionNode` | `name`, `type`, `defaultValue`, `directives` |
| `SCALAR_TYPE_DEFINITION` | `ScalarTypeDefinitionNode` | `name`, `directives` |
| `DIRECTIVE_DEFINITION` | `DirectiveDefinitionNode` | `name`, `arguments`, `locations` |
| `SCHEMA_EXTENSION` | `SchemaExtensionNode` | `operationTypes`, `directives` |
| `OBJECT_TYPE_EXTENSION` | `ObjectTypeExtensionNode` | `name`, `fields`, `interfaces`, `directives` |
| `INTERFACE_TYPE_EXTENSION` | `InterfaceTypeExtensionNode` | `name`, `fields`, `interfaces` |
| `UNION_TYPE_EXTENSION` | `UnionTypeExtensionNode` | `name`, `types` |
| `ENUM_TYPE_EXTENSION` | `EnumTypeExtensionNode` | `name`, `values` |
| `INPUT_OBJECT_TYPE_EXTENSION` | `InputObjectTypeExtensionNode` | `name`, `fields` |
| `SCALAR_TYPE_EXTENSION` | `ScalarTypeExtensionNode` | `name`, `directives` |
| `NAMED_TYPE` | `NamedTypeNode` | `name` |
| `LIST_TYPE` | `ListTypeNode` | `type` |
| `NON_NULL_TYPE` | `NonNullTypeNode` | `type` |

---

### 5. Operation AST — node kinds and key properties

| Kind constant | Node type | Key properties |
|---|---|---|
| `DOCUMENT` | `DocumentNode` | `definitions` |
| `OPERATION_DEFINITION` | `OperationDefinitionNode` | `operation` (`"query"` \| `"mutation"` \| `"subscription"`), `name`, `variableDefinitions`, `selectionSet`, `directives` |
| `FRAGMENT_DEFINITION` | `FragmentDefinitionNode` | `name`, `typeCondition`, `selectionSet`, `directives` |
| `SELECTION_SET` | `SelectionSetNode` | `selections` |
| `FIELD` | `FieldNode` | `alias`, `name`, `arguments`, `selectionSet`, `directives` |
| `FRAGMENT_SPREAD` | `FragmentSpreadNode` | `name`, `directives` |
| `INLINE_FRAGMENT` | `InlineFragmentNode` | `typeCondition`, `selectionSet`, `directives` |
| `VARIABLE_DEFINITION` | `VariableDefinitionNode` | `variable`, `type`, `defaultValue`, `directives` |
| `ARGUMENT` | `ArgumentNode` | `name`, `value` |
| `DIRECTIVE` | `DirectiveNode` | `name`, `arguments` |
| `INT` | `IntValueNode` | `value` |
| `FLOAT` | `FloatValueNode` | `value` |
| `STRING` | `StringValueNode` | `value`, `block` |
| `BOOLEAN` | `BooleanValueNode` | `value` |
| `NULL` | `NullValueNode` | — |
| `ENUM` | `EnumValueNode` | `value` |
| `LIST` | `ListValueNode` | `values` |
| `OBJECT` | `ObjectValueNode` | `fields` |
| `OBJECT_FIELD` | `ObjectFieldNode` | `name`, `value` |
| `VARIABLE` | `VariableNode` | `name` |

---

### 6. Node nesting structure and traversal order

Understanding which nodes nest inside which — and in what order — is essential for knowing where in the ancestor chain you are, what `return false` will skip, and when `BREAK` fires.

#### Operation AST

```
DocumentNode
└── definitions[]
    ├── OperationDefinitionNode
    │   ├── name?              → NameNode
    │   ├── variableDefinitions[] → VariableDefinitionNode
    │   │   ├── variable       → VariableNode → NameNode
    │   │   ├── type           → NamedTypeNode | ListTypeNode | NonNullTypeNode
    │   │   └── defaultValue?  → <ValueNode>
    │   ├── directives[]       → DirectiveNode
    │   │   └── arguments[]    → ArgumentNode → name, value → <ValueNode>
    │   └── selectionSet       → SelectionSetNode
    │       └── selections[]
    │           ├── FieldNode
    │           │   ├── alias?        → NameNode
    │           │   ├── name          → NameNode
    │           │   ├── arguments[]   → ArgumentNode → value → <ValueNode>
    │           │   ├── directives[]  → DirectiveNode
    │           │   └── selectionSet? → SelectionSetNode  ← (recurse)
    │           ├── InlineFragmentNode
    │           │   ├── typeCondition? → NamedTypeNode
    │           │   ├── directives[]
    │           │   └── selectionSet   → SelectionSetNode  ← (recurse)
    │           └── FragmentSpreadNode
    │               ├── name          → NameNode
    │               └── directives[]
    └── FragmentDefinitionNode
        ├── name           → NameNode
        ├── typeCondition  → NamedTypeNode
        ├── variableDefinitions[]
        ├── directives[]
        └── selectionSet   → SelectionSetNode  ← (recurse)
```

`<ValueNode>`: `IntValueNode`, `FloatValueNode`, `StringValueNode`, `BooleanValueNode`, `NullValueNode`, `EnumValueNode`, `VariableNode`, `ListValueNode` (→ `values[]`), `ObjectValueNode` (→ `fields[]` → `ObjectFieldNode` → `value`).

`IntValueNode`, `FloatValueNode`, `StringValueNode`, `BooleanValueNode`, `NullValueNode`, `EnumValueNode`, and `NameNode` are **leaves** — `visit()` calls their handler but does not recurse further.

#### Schema AST

```
DocumentNode
└── definitions[]
    ├── SchemaDefinitionNode
    │   ├── directives[]
    │   └── operationTypes[] → OperationTypeDefinitionNode → type → NamedTypeNode
    ├── ObjectTypeDefinitionNode
    │   ├── name         → NameNode
    │   ├── interfaces[] → NamedTypeNode
    │   ├── directives[] → DirectiveNode
    │   └── fields[]     → FieldDefinitionNode
    │       ├── name         → NameNode
    │       ├── arguments[]  → InputValueDefinitionNode
    │       ├── type         → NamedTypeNode | ListTypeNode | NonNullTypeNode
    │       └── directives[] → DirectiveNode
    ├── InterfaceTypeDefinitionNode  (same shape as ObjectType)
    ├── UnionTypeDefinitionNode
    │   ├── name, directives[]
    │   └── types[]      → NamedTypeNode
    ├── EnumTypeDefinitionNode
    │   ├── name, directives[]
    │   └── values[]     → EnumValueDefinitionNode → name, directives[]
    ├── InputObjectTypeDefinitionNode
    │   ├── name, directives[]
    │   └── fields[]     → InputValueDefinitionNode
    │       ├── name          → NameNode
    │       ├── type          → NamedTypeNode | ListTypeNode | NonNullTypeNode
    │       ├── defaultValue? → <ValueNode>
    │       └── directives[]  → DirectiveNode
    ├── ScalarTypeDefinitionNode    → name, directives[]
    ├── DirectiveDefinitionNode
    │   ├── name, locations[]  (NameNode leaves)
    │   └── arguments[]    → InputValueDefinitionNode
    └── *TypeExtensionNode  (mirrors the corresponding definition above)
```

Type wrapper recursion: `NonNullTypeNode.type` → `NamedTypeNode | ListTypeNode`; `ListTypeNode.type` → any type node (can be deeply nested for `[String!]!`).

#### `QueryDocumentKeys` — the traversal map `visit()` uses internally

`visit()` consults this object to determine which properties of each node contain child nodes and in what order to traverse them. The key order defines traversal order within a node.

```js
// From graphql/language — only properties containing child ASTNodes are listed
const QueryDocumentKeys = {
  Document:                  ['definitions'],
  OperationDefinition:       ['name', 'variableDefinitions', 'directives', 'selectionSet'],
  VariableDefinition:        ['variable', 'type', 'defaultValue', 'directives'],
  Variable:                  ['name'],
  SelectionSet:              ['selections'],
  Field:                     ['alias', 'name', 'arguments', 'directives', 'selectionSet'],
  Argument:                  ['name', 'value'],
  FragmentSpread:            ['name', 'directives'],
  InlineFragment:            ['typeCondition', 'directives', 'selectionSet'],
  FragmentDefinition:        ['name', 'variableDefinitions', 'typeCondition', 'directives', 'selectionSet'],
  Directive:                 ['name', 'arguments'],
  NamedType:                 ['name'],
  ListType:                  ['type'],
  NonNullType:               ['type'],
  ListValue:                 ['values'],
  ObjectValue:               ['fields'],
  ObjectField:               ['name', 'value'],
  SchemaDefinition:          ['description', 'directives', 'operationTypes'],
  OperationTypeDefinition:   ['type'],
  ObjectTypeDefinition:      ['description', 'name', 'interfaces', 'directives', 'fields'],
  FieldDefinition:           ['description', 'name', 'arguments', 'type', 'directives'],
  InputValueDefinition:      ['description', 'name', 'type', 'defaultValue', 'directives'],
  InterfaceTypeDefinition:   ['description', 'name', 'interfaces', 'directives', 'fields'],
  UnionTypeDefinition:       ['description', 'name', 'directives', 'types'],
  EnumTypeDefinition:        ['description', 'name', 'directives', 'values'],
  EnumValueDefinition:       ['description', 'name', 'directives'],
  InputObjectTypeDefinition: ['description', 'name', 'directives', 'fields'],
  DirectiveDefinition:       ['description', 'name', 'arguments', 'locations'],
  SchemaExtension:           ['directives', 'operationTypes'],
  ObjectTypeExtension:       ['name', 'interfaces', 'directives', 'fields'],
  InterfaceTypeExtension:    ['name', 'interfaces', 'directives', 'fields'],
  UnionTypeExtension:        ['name', 'directives', 'types'],
  EnumTypeExtension:         ['name', 'directives', 'values'],
  InputObjectTypeExtension:  ['name', 'directives', 'fields'],
  ScalarTypeExtension:       ['name', 'directives'],
};
```

> Nodes absent from this map (`IntValue`, `FloatValue`, `StringValue`, `BooleanValue`, `NullValue`, `EnumValue`, `Name`) are **leaves** — `visit()` fires their visitor handler but never recurses into them.

**Custom traversal via the 3rd `visitorKeys` argument:**

Pass a partial override to restrict descent or add custom node kinds:

```js
visit(myDoc, myVisitor, {
  // Only descend into fields for ObjectTypeDefinition, skip directives
  ObjectTypeDefinition: ['fields'],
  // Teach visit() about a custom node kind
  MyCustomNode: ['left', 'right'],
});
```

---

### 7. Practical examples

**Collect all field names declared in a schema**

```js
import { visit } from 'graphql/language';

const fieldNames = [];
visit(schemaDoc, {
  FieldDefinition(node) {
    fieldNames.push(node.name.value);
  }
});
```

**Find all types implementing an interface**

```js
import { visit } from 'graphql/language';

const implementors = [];
visit(schemaDoc, {
  ObjectTypeDefinition(node) {
    const implementsNode = node.interfaces?.some(i => i.name.value === 'Node');
    if (implementsNode) implementors.push(node.name.value);
  }
});
```

**Collect all directives used in a schema (with their locations)**

```js
import { visit } from 'graphql/language';

const usages = [];
visit(schemaDoc, {
  Directive(node, _key, _parent, _path, ancestors) {
    const parentNode = ancestors[ancestors.length - 1];
    usages.push({ directive: node.name.value, onKind: parentNode.kind });
  }
});
```

**Collect all variables in an operation document**

```js
import { visit, print } from 'graphql/language';

const vars = [];
visit(operationDoc, {
  VariableDefinition(node) {
    vars.push({ name: node.variable.name.value, type: print(node.type) });
  }
});
```

**Rename a field in operations (AST rewrite)**

```js
import { visit } from 'graphql/language';

const rewritten = visit(operationDoc, {
  Field(node) {
    if (node.name.value === 'oldName') {
      return { ...node, name: { ...node.name, value: 'newName' } };
    }
  }
});
```

**Remove all `@deprecated` directives from a schema AST**

```js
import { visit } from 'graphql/language';

const cleaned = visit(schemaDoc, {
  Directive(node) {
    if (node.name.value === 'deprecated') return null;
  }
});
```

**Early exit with `BREAK`**

```js
import { visit, BREAK } from 'graphql/language';

let hasSubscription = false;
visit(operationDoc, {
  OperationDefinition(node) {
    if (node.operation === 'subscription') {
      hasSubscription = true;
      return BREAK;
    }
  }
});
```

**Skip a subtree on enter**

```js
import { visit } from 'graphql/language';

// Don't descend into __schema or __type introspection fields
visit(operationDoc, {
  Field: {
    enter(node) {
      if (node.name.value.startsWith('__')) return false;
    }
  }
});
```

---

### 8. Review checklist

When reviewing an existing visitor, verify each of the following:

- [ ] **Kind keys are correct** — misspelled kinds (e.g. `FieldDefinitions` vs `FieldDefinition`) silently match nothing; no error is thrown
- [ ] **`enter` vs `leave` matches intent** — use `enter` for skipping subtrees (`return false`), use `leave` for bottom-up computation where child values are needed
- [ ] **Return value is intentional** — omitting `return` in a modifier visitor is a silent no-op; every branch that intends to replace/remove must explicitly return
- [ ] **Null safety on optional properties** — `node.selectionSet`, `node.alias`, `node.defaultValue`, `node.name` (on anonymous operations) may be `undefined`
- [ ] **`visitInParallel` caveat understood** — parallel visitors see the original node, not edits made by sibling visitors
- [ ] **`BREAK` used for early exit** — avoid manually short-circuiting by wrapping visit in try/catch with a sentinel throw; use `BREAK` instead
- [ ] **No schema / operation kind confusion** — `FieldDefinition` (schema) vs `Field` (operation) are different Kinds; mixing them in a visitor targeting the wrong document type does nothing
- [ ] **`path` and `ancestors` usage is correct** — `ancestors[ancestors.length - 1]` is the immediate parent node; `ancestors` contains `ASTNode` objects OR `readonly ASTNode[]` arrays (when the parent is an array property like `fields`)
- [ ] **`visit()` return value is used when rewriting** — the function returns a new root; mutations do not apply if the result is discarded

---

## Key constraints

- Import `visit`, `BREAK`, `visitInParallel` from `'graphql/language'`, not from the `'graphql'` root, for tree-shaking
- Visitor functions are **synchronous** — `async`/`await` inside a visitor function silently breaks the return-value contract (`Promise` is treated as a replacement node)
- `visit()` does **not** mutate the input AST — always capture the return value when rewriting
- Modifying the node returned from `enter` does not affect the `leave` call for the same node — `leave` always receives the traversal-time original
- To convert a `GraphQLSchema` to an AST, use `parse(printSchema(schema))` — do not use `introspectionFromSchema`, which returns a non-AST introspection result object
