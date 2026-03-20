# How to Use tofu graph to Visualize Dependencies

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use tofu graph to generate a dependency graph of your infrastructure resources and visualize it with Graphviz or other tools.

## Introduction

`tofu graph` outputs a dependency graph of your OpenTofu configuration in DOT format. Visualizing dependencies helps you understand resource creation order, debug dependency issues, and document your infrastructure architecture.

## Basic Usage

```bash
# Generate the dependency graph in DOT format
tofu graph

# The output is DOT language (Graphviz format)
# digraph {
#   compound = "true"
#   newrank  = "true"
#   subgraph "root" {
#     "[root] aws_instance.web (expand)" -> ...
```

## Rendering with Graphviz

```bash
# Install Graphviz
# macOS:
brew install graphviz

# Ubuntu/Debian:
apt-get install graphviz

# Generate a PNG image
tofu graph | dot -Tpng -o infrastructure.png

# Generate an SVG (better for large graphs)
tofu graph | dot -Tsvg -o infrastructure.svg

# Open the image
open infrastructure.png  # macOS
xdg-open infrastructure.png  # Linux
```

## Graph Types

```bash
# Plan graph (default) — shows planned operations
tofu graph -type=plan

# Apply graph — shows apply operations
tofu graph -type=apply

# Destroy graph — shows destruction order
tofu graph -type=destroy-complete

# Plan with destroy — destroy plan graph
tofu graph -type=plan-destroy
```

## Filtering with -draw-cycles

```bash
# Draw dependency cycles (helps debug circular dependencies)
tofu graph -draw-cycles | dot -Tpng -o cycles.png
```

## Simplifying Large Graphs

For large configurations, the graph can be complex. Filter and simplify:

```bash
# Focus on a specific resource's dependencies
tofu graph | grep -A5 "aws_eks_cluster"

# Use unflatten to make large graphs more readable
tofu graph | unflatten -l 10 | dot -Tsvg -o infrastructure.svg
```

## Using blast2dot for Better Visualization

For better-looking graphs:

```bash
# Install blast2dot
pip install blast2dot

# Generate a cleaner graph
tofu graph | blast2dot | dot -Tsvg -o clean-graph.svg
```

## Online DOT Visualization

If you can't install Graphviz locally:

```bash
# Generate DOT output
tofu graph > infrastructure.dot

# Open https://dreampuf.github.io/GraphvizOnline/
# Paste the .dot file content
```

## Understanding the Graph Output

```
digraph {
  # Node: represents a resource
  "[root] aws_instance.web (expand)" [label = "aws_instance.web", ...]

  # Edge: dependency arrow (source depends on destination)
  "[root] aws_instance.web (expand)" -> "[root] aws_security_group.web (expand)"
  #  aws_instance.web depends on aws_security_group.web (sg created first)
}
```

Arrow direction: `A -> B` means A depends on B (B is created first).

## Module Dependency Graphs

```bash
# Graph including module internals
tofu graph -module-depth=2

# Limit module depth (cleaner for deeply nested configs)
tofu graph -module-depth=1 | dot -Tsvg -o modules.svg
```

## Practical Example

```bash
# Generate a comprehensive visualization

# Step 1: Generate DOT
tofu graph -type=plan > plan-graph.dot

# Step 2: Render as SVG
dot -Tsvg plan-graph.dot -o plan-graph.svg

# Step 3: Open in browser
python3 -m http.server 8080 &
echo "View at: http://localhost:8080/plan-graph.svg"
```

## Debugging Circular Dependencies

When you see `Error: Cycle: ...`:

```bash
# Generate the graph to see the cycle
tofu graph -draw-cycles | dot -Tpng -o cycle.png

# The cycle will be highlighted in the graph
# Common causes:
# - Two resources referencing each other
# - Module output referenced by its own input
```

## Integrating into Documentation

```bash
# Generate graph for documentation
tofu graph | dot -Tsvg > docs/infrastructure-dependency-graph.svg
```

## Conclusion

`tofu graph` is a powerful tool for understanding and documenting infrastructure dependencies. Use it to debug dependency-related errors, understand creation order, and create architecture documentation. For complex infrastructures, filter the graph to focus on specific subsystems and use SVG format for better quality in documentation.
