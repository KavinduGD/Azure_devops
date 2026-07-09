Here's a concise summary of the article **"Set variables in scripts"** from the Azure DevOps documentation.

## Purpose

Azure DevOps allows you to create variables **during pipeline execution** from a Bash or PowerShell script using a **logging command**.

This is useful when:

- You calculate a value during the pipeline (timestamp, commit hash, version number, etc.).
- You call an API and store its response.
- You want later tasks to use a value created by an earlier task.

---

## Basic Syntax

In Bash:

```bash
echo "##vso[task.setvariable variable=myVar;]foo"
```

This creates a variable named `myVar` with the value:

```
foo
```

Later tasks in the **same job** can access it using:

```yaml
$(myVar)
```

---

## Important Rule

A variable created with `task.setvariable` **cannot be used in the same script** that creates it.

Example:

```yaml
- bash: |
    echo "##vso[task.setvariable variable=name]John"
    echo $(name)    # ❌ Doesn't work here
```

It becomes available only in **subsequent tasks**.

---

# Variable Properties

You can add optional properties.

| Property          | Purpose                              |
| ----------------- | ------------------------------------ |
| `variable=`       | Variable name (required)             |
| `isSecret=true`   | Hide value in logs                   |
| `isOutput=true`   | Allow sharing with other jobs/stages |
| `isReadOnly=true` | Prevent later tasks from changing it |

Example:

```bash
echo "##vso[task.setvariable variable=TOKEN;isSecret=true]abc123"
```

---

# Secret Variables

Using

```bash
issecret=true
```

means:

- value is stored securely
- logs display `***`
- protects passwords/API keys

Example:

```bash
echo "##vso[task.setvariable variable=password;issecret=true]mypassword"
```

---

# Output Variables

There are **four levels**.

## 1. Same Job (Normal Variable)

```bash
echo "##vso[task.setvariable variable=BUILD_DATE]2026"
```

Use later:

```yaml
$(BUILD_DATE)
```

---

## 2. Same Job (`isOutput=true`)

If you specify

```bash
isOutput=true
```

you **must name the task**.

Example

```yaml
- bash: |
    echo "##vso[task.setvariable variable=version;isOutput=true]1.0"
  name: BuildInfo
```

Access:

```yaml
$(BuildInfo.version)
```

---

## 3. Future Job

Job A

```yaml
- bash: |
    echo "##vso[task.setvariable variable=image;isOutput=true]v1"
  name: SetImage
```

Job B

```yaml
dependsOn: A

variables:
  imageTag: $[ dependencies.A.outputs['SetImage.image'] ]
```

Now

```yaml
$(imageTag)
```

contains the value.

---

## 4. Future Stage

Stage A

```yaml
echo "##vso[task.setvariable variable=version;isOutput=true]1.0"
```

Stage B

```yaml
variables:
  version: $[
    stageDependencies.A.A1.outputs['MyTask.version']
  ]
```

This lets information flow across stages.

---

# Read-Only Variables

```bash
echo "##vso[task.setvariable variable=VERSION;isReadOnly=true]1.0"
```

Later tasks cannot overwrite it.

Useful for:

- build numbers
- commit IDs
- release versions

---

# Escaping Newlines

If a variable contains multiple lines, special characters must be escaped:

- `%` → `%AZP25`
- newline → `%0A`
- carriage return → `%0D`

Azure DevOps automatically restores them when reading the variable.

---

# Troubleshooting

If a variable doesn't work:

- Don't try to use it in the same task where it was created.
- Ensure `dependsOn` is set when sharing across jobs or stages.
- If using `isOutput=true`, reference it with the task name.
- Check for accidental spaces around `isOutput=true`.
- Print all environment variables with:

```yaml
- script: env
```

or inspect dependencies:

```yaml
variables:
  deps: $[ convertToJson(dependencies) ]
```

---

# Connection to Your Pipeline

In your pipeline, you used:

```bash
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
COMMIT=$(git rev-parse --short HEAD)

echo "##vso[task.setvariable variable=BUILD_DATE]$TIMESTAMP"
echo "##vso[task.setvariable variable=GIT_COMMIT]$COMMIT"
echo "##vso[task.setvariable variable=DOCKER_TAG]dev-$COMMIT-$TIMESTAMP"
```

This creates three variables:

- `BUILD_DATE` → build timestamp
- `GIT_COMMIT` → short Git commit hash
- `DOCKER_TAG` → combined Docker image tag (e.g., `dev-a1b2c3d-20260709-083000`)

Later tasks in the same job can use them directly:

```yaml
tags: |
  $(DOCKER_TAG)
```

or

```bash
echo $(BUILD_DATE)
```

This is one of the most common Azure DevOps patterns for generating dynamic Docker image tags and passing calculated values between pipeline steps.
