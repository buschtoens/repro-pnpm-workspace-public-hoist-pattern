# `pnpm` 🪳: `public-hoist-pattern` ignores workspaces packages

**Expected behavior:** Packages that match the `public-hoist-pattern` are
hoisted to the root `node_modules`, *including packages from the workspace
itself*.

**Actual behavior:** Only external dependencies and workspace-internal
dependencies *listed as root dependencies* are hoisted.

## Demonstration

```sh
# Create `.npmrc` and install dependencies, by using either one of:

# empty → Default `pnpm`-internal fallback will apply.
$ pnpm npmrc:relink empty

# defaults → Same as above, but explicitly set the defaults, instead of fallback.
$ pnpm npmrc:relink defaults

# explicit → Like `defaults` + explicit enumeration of all `eslint-config-my-*`.
$ pnpm npmrc:relink explicit

# Verify value of `public-hoist-pattern`
$ pnpm npmrc:get
# empty    → `undefined`
# defaults → `*types*,*eslint*,@prettier/plugin-*,*prettier-plugin-*`
# explicit → `*types*,*eslint*,@prettier/plugin-*,*prettier-plugin-*,eslint-config-my-example-a,eslint-config-my-example-b,eslint-config-my-example-c,eslint-config-my-prettier
```

```sh
# Then pick an `.eslintrc.js` scenario and run it:

# my-prettier → ✅ hoisted, due to reference in root `package.json`
$ pnpm lint:my-prettier

# my-example-a → ✅ hoisted, due to reference in root `package.json`
$ pnpm lint:my-example-a

# my-example-b → ❌ NOT hoisted, but should be
$ pnpm lint:my-example-b

# my-example-c → ✅ hoisted, due to reference in root `package.json`
$ pnpm lint:my-example-c
```

## Dependency Setup

**Workspaces packages:**

- `eslint-config-my-example-a`: only hoisted due to being referenced in root `package.json`
- `eslint-config-my-example-b`: not hoisted, but should be
- `eslint-config-my-example-c`: only hoisted due to being referenced in root `package.json`
- `eslint-config-my-prettier`: not hoisted, but should be
- `my-prettier-config`: only hoisted due to being referenced in root `package.json`
- `my-prettier-config`: shouldn't normally be hoisted, but is hoisted due to
  being referenced in root `package.json`

**Dependency chain:**

- `eslint-config-my-example-c` →
  - `eslint-config-my-example-b` →
    - `eslint-config-my-example-a` →
      - `eslint-config-my-prettier` →
        - `my-prettier-config`

### Workspace Root `package.json`

```js
{ // ...
  "devDependencies": {
    "eslint": "^7.32.0",
    "eslint-config-my-example-a": "workspace:*",
    "eslint-config-my-example-c": "workspace:*",
    "eslint-config-my-prettier": "workspace:*",
    "my-prettier-config": "workspace:*"
  }
}
```

### `my-prettier-config`

No dependencies. Exports a `prettier` config.

### `eslint-config-my-prettier`

```js
{ // ...
  "dependencies": {
    "eslint-config-prettier": "^8.3.0", // external, hoisted
    "eslint-plugin-prettier": "^3.4.0", // external, hoisted
    "prettier": "^2.3.2" // external, hoisted
  }
}
```

#### `eslint-config-my-example-*`

#### `eslint-config-my-example-a`

```js
{ // ...
  "dependencies": {
    "eslint-config-my-prettier": "workspace:^"
  }
}
```

#### `eslint-config-my-example-b`

```js
{ // ...
  "dependencies": {
    "eslint-config-my-example-a": "workspace:^"
  }
}
```

#### `eslint-config-my-example-c`

```js
{ // ...
  "dependencies": {
    "eslint-config-my-example-b": "workspace:^"
  }
}
```


## Scripts

### `.eslintrc.js`

#### `pnpm lint`

Invoke `eslint` with the current `.eslintrc.js`.

```sh
eslint .
```

#### `pnpm lint:relink <name>`

Links `.eslintrc.js` to `.<name>.eslintrc.js` and then calls `pnpm lint`.

These configs are available:

#### `pnpm lint:my-prettier` / `pnpm lint:relink my-prettier`

```js
module.exports = {
  extends: 'my-prettier',
};
```

#### `pnpm lint:my-example-a` / `pnpm lint:relink my-example-a`

```js
module.exports = {
  extends: 'my-example-a',
};
```

#### `pnpm lint:my-example-b` / `pnpm lint:relink my-example-b`

```js
module.exports = {
  extends: 'my-example-b',
};
```

#### `pnpm lint:my-example-c` / `pnpm lint:relink my-example-c`

```js
module.exports = {
  extends: 'my-example-c',
};
```

### `.npmrc`

#### `pnpm npmrc:get`

Prints the current config value of `public-hoist-pattern`.

```sh
pnpm config get public-hoist-pattern
```

#### `pnpm npmrc:relink <name>`

Links `.npmrc` to `.<name>.npmrc`.

These configs are available:

##### `pnpm npmrc:relink empty`

```yml
# .empty.npmrc.js
#
# No entries.
```

##### `pnpm npmrc:relink defaults`

```yml
# .defaults.npmrc.js
#
# Repeat default `public-hoist-pattern`.
#
# https://pnpm.io/npmrc#public-hoist-pattern
# https://github.com/pnpm/pnpm/blob/v6.11.5/packages/config/src/index.ts#L180-L187
public-hoist-pattern[]='*types*'
public-hoist-pattern[]='*eslint*'
public-hoist-pattern[]='@prettier/plugin-*'
public-hoist-pattern[]='*prettier-plugin-*'
```

##### `pnpm npmrc:relink explicit`

```yml
# .explicit.npmrc.js
#
# Repeat default `public-hoist-pattern` and add explicit patterns on top.

# Defaults:
# https://pnpm.io/npmrc#public-hoist-pattern
# https://github.com/pnpm/pnpm/blob/v6.11.5/packages/config/src/index.ts#L180-L187
public-hoist-pattern[]='*types*'
public-hoist-pattern[]='*eslint*'
public-hoist-pattern[]='@prettier/plugin-*'
public-hoist-pattern[]='*prettier-plugin-*'

# Explicit additions:
public-hoist-pattern[]='eslint-config-my-example-a'
public-hoist-pattern[]='eslint-config-my-example-b'
public-hoist-pattern[]='eslint-config-my-example-c'
public-hoist-pattern[]='eslint-config-my-prettier'
```

### Dependencies

#### `pnpm deps:hoisted`

Lists the hoisted dependencies.

```sh
ls node_modules
```

#### `pnpm deps:clean`

Wipes all `node_modules` and deletes `pnpm-lock.yml`.

```sh
pnpm --recursive exec rm -rf ./node_modules && rm -f pnpm-lock.yaml
```

#### `pnpm deps:reinstall`

Wipes all `node_modules`, deletes `pnpm-lock.yml` and then reinstalls.

```sh
pnpm deps:clean && pnpm install
```
