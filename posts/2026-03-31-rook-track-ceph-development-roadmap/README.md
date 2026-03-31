# How to Track Ceph Development Roadmap

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Ceph, Rook, Roadmap, Development, Community

Description: Track the Ceph development roadmap using the Ceph tracker, mailing lists, release notes, and community meetings to plan upgrades and anticipate new features.

---

Staying informed about Ceph's development direction helps you plan upgrades, prepare for deprecations, and take advantage of new features before they are widely documented.

## Ceph Tracker Roadmap

The primary source for roadmap information is the Ceph issue tracker:

- https://tracker.ceph.com/projects/ceph/roadmap

Filter by release to see what is planned for the next version:

```
# Example URL for Squid (19) roadmap
https://tracker.ceph.com/projects/ceph/roadmap?completed=0
```

Click on individual features to see the design document link and implementation status.

## GitHub - Merge Activity

Watch the Ceph GitHub repository to track active development:

- https://github.com/ceph/ceph

Use the releases page to see what changed between versions:

```bash
# Changelog between two tags
# https://github.com/ceph/ceph/compare/v18.2.1...v18.2.2
```

## Watching Release Notes

Ceph publishes detailed release notes for every version:

- https://docs.ceph.com/en/latest/releases/

Subscribe to ceph-announce@ceph.io to receive notifications when new versions are released:

```
Subscribe: https://lists.ceph.io/postorius/lists/ceph-announce.ceph.io/
```

## Attending Ceph Community Calls

The Ceph development community holds bi-weekly calls:

- Ceph Developer Summit: held before each major release
- Ceph Leadership Team meetings: open to the public, schedule at ceph.io/community

## Rook Roadmap

If you use Rook-Ceph, track the Rook roadmap separately:

- https://github.com/rook/rook/milestones

Rook typically adds support for new Ceph versions within a few weeks of their GA release.

## Tracking Deprecations

Subscribe to the ceph-users list and watch the release notes for entries tagged "deprecated":

```bash
# Search release notes for deprecations
curl -s https://docs.ceph.com/en/latest/_sources/releases/reef.rst.txt | grep -i "deprecat"
```

## Setting Up Notifications

Use GitHub's watch feature to get notified of new releases:

1. Visit https://github.com/ceph/ceph
2. Click "Watch" > "Custom" > select "Releases"

For Rook:

1. Visit https://github.com/rook/rook
2. Click "Watch" > "Custom" > select "Releases"

## Summary

Tracking the Ceph roadmap involves monitoring the issue tracker, watching GitHub releases, subscribing to announcement mailing lists, and attending community calls. For Rook users, tracking both repositories ensures you catch compatibility requirements between Rook and new Ceph versions before they affect your upgrade planning.
