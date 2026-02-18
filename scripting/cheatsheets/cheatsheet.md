# Scripting Cheatsheet

> Quick-reference for Bash and Python scripting.

## Bash

### Script Boilerplate
```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'

# Usage: ./script.sh <arg1>
```

### Variables & Arrays
```bash
NAME="world"
echo "Hello, $NAME"

ARRAY=("one" "two" "three")
echo "${ARRAY[0]}"        # one
echo "${#ARRAY[@]}"       # length: 3
```

### Conditionals
```bash
if [[ -f "$FILE" ]]; then
  echo "file exists"
elif [[ -d "$FILE" ]]; then
  echo "directory"
else
  echo "not found"
fi
```

### Loops
```bash
for item in a b c; do echo "$item"; done

while IFS= read -r line; do
  echo "$line"
done < input.txt
```

### Functions
```bash
greet() {
  local name="$1"
  echo "Hello, $name"
}
greet "Alice"
```

### Useful One-Liners
```bash
# Find and replace in files
sed -i 's/old/new/g' file.txt

# Process CSV
awk -F',' '{print $1, $3}' data.csv

# Count lines matching pattern
grep -c "ERROR" app.log
```

## Python

### File I/O
```python
with open("file.txt") as f:
    for line in f:
        print(line.strip())
```

### JSON & YAML
```python
import json, yaml

data = json.loads(open("data.json").read())
config = yaml.safe_load(open("config.yaml"))
```

### subprocess
```python
import subprocess
result = subprocess.run(["ls", "-la"], capture_output=True, text=True)
print(result.stdout)
```

### requests (API calls)
```python
import requests
r = requests.get("https://api.example.com/data")
print(r.json())
```

## Common Patterns

<!-- Add error handling patterns, retry loops, logging setup here -->
