# How to Use LOLWUT in Redis (Easter Egg and Version Check)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Debugging, Server

Description: Explore the Redis LOLWUT command - an Easter egg that displays artistic output along with the Redis version for quick version checks.

---

## What Is LOLWUT?

`LOLWUT` is an official Redis command introduced in Redis 5.0 as a playful easter egg. It renders a piece of ASCII art in the terminal and appends the current Redis server version. Each major Redis version ships with different artwork.

Despite being whimsical, `LOLWUT` is a fully supported, documented command with practical uses.

## Basic Usage

```bash
redis-cli LOLWUT
```

Example output (Redis 7.x):

```text
^    ^    ^
|    |    |
|    |    |
|    |    |

Redis ver. 7.2.0
```

The art varies by version and may include fractals, maze patterns, or geometric shapes.

## Customizing the Output

Some versions accept parameters to adjust the artwork:

```bash
redis-cli LOLWUT VERSION 5
```

This requests the artwork from a specific Redis version (useful when you want to see what older art looked like).

```bash
redis-cli LOLWUT 10
```

Some versions accept a numeric argument to change the complexity or scale of the artwork.

## Practical Use: Quick Version Check

The most practical reason to use `LOLWUT` is the version line appended to the output:

```bash
redis-cli LOLWUT | tail -1
```

Output:

```text
Redis ver. 7.2.0
```

This is a quick way to confirm the Redis version without parsing `INFO server` output.

Compared to the standard version check:

```bash
redis-cli INFO server | grep redis_version
```

`LOLWUT | tail -1` is slightly more fun, though both work equally well.

## Using in Health Check Scripts

```bash
#!/bin/bash
VERSION=$(redis-cli LOLWUT | tail -1 | grep -oE '[0-9]+\.[0-9]+\.[0-9]+')
echo "Redis version: $VERSION"
```

## Each Redis Version Has Different Art

| Redis Version | Artwork Description |
|--------------|---------------------|
| 5.0 | Dragon curve fractal |
| 6.0 | Mandelbrot set |
| 6.2 | 3D hilbert curve |
| 7.0 | Spinning shapes |
| 7.2 | Various geometric patterns |

## Why Redis Added LOLWUT

The Redis creator antirez added `LOLWUT` as a nod to the open-source culture of creating fun things. It demonstrates that infrastructure software does not have to be purely utilitarian.

The command also serves as a living artifact - each Redis release ships with new artwork created by contributors, making it part of the release history.

## Summary

`LOLWUT` is Redis's official easter egg that generates ASCII art and prints the server version. While whimsical by nature, it serves as a quick version check tool and a reminder that engineering culture includes creativity. Run it whenever you want a quick version confirmation with a side of artistry.
