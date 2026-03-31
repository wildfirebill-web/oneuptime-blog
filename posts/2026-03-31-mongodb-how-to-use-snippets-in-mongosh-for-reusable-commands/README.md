# How to Use Snippets in mongosh for Reusable Commands

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Mongosh, Snippet, Reusability, Shell, Automation

Description: Learn how to use mongosh snippets to create reusable command libraries that can be shared across sessions and team members.

---

## Introduction

mongosh snippets are reusable JavaScript packages that extend the shell with custom commands and utilities. They are similar to npm packages scoped to the shell environment. Snippets allow teams to standardize common operations - like health checks, index audits, or data exports - into shareable libraries.

## Enabling the Snippets Feature

Snippets require the snippet registry to be configured. By default, mongosh uses the MongoDB-hosted registry:

```javascript
snippet install mongocompat
```

List all available snippets:

```javascript
snippet ls
```

## Installing a Snippet

Install a snippet by name. For example, the `mongocompat` snippet adds compatibility helpers:

```bash
snippet install mongocompat
```

After installation, load it in your session:

```javascript
load("mongocompat");
```

## Writing a Custom Snippet

Create a directory structure for your snippet package:

```bash
mkdir my-snippets
cd my-snippets
npm init -y
```

Write the snippet logic in `index.js`:

```javascript
// index.js
function dbStats(dbName) {
  const db = db.getSiblingDB(dbName);
  const stats = db.stats();
  print(`Database: ${dbName}`);
  print(`Collections: ${stats.collections}`);
  print(`Data size: ${(stats.dataSize / 1024 / 1024).toFixed(2)} MB`);
  print(`Index size: ${(stats.indexSize / 1024 / 1024).toFixed(2)} MB`);
}

function slowQueries(threshold = 100) {
  const ops = db.currentOp({ secs_running: { $gt: threshold / 1000 } });
  return ops.inprog.map(op => ({
    opid: op.opid,
    ns: op.ns,
    secs: op.secs_running,
    op: op.op
  }));
}

module.exports = { dbStats, slowQueries };
```

## Using a Local Snippet

Load a local file directly in mongosh:

```javascript
load("/path/to/my-snippets/index.js");
dbStats("production");
slowQueries(500);
```

## Persisting Snippets in .mongoshrc.js

Add frequently used snippets to your `~/.mongoshrc.js` file so they load automatically:

```javascript
// ~/.mongoshrc.js
load("/opt/mongosh-snippets/utils.js");
print("Custom utilities loaded");
```

## Sharing Snippets with Your Team

Publish snippets to a private npm registry so the team can install them:

```bash
npm publish --registry https://registry.mycompany.com
```

Then team members install with:

```javascript
snippet install @mycompany/mongo-utils
```

## Summary

mongosh snippets provide a structured way to build and share reusable MongoDB utilities. By packaging common operations as snippet modules, teams can standardize workflows, reduce repetition, and onboard new members more quickly. Loading snippets from `.mongoshrc.js` ensures your tools are always available when you open the shell.
