# How to Understand Mobile IPv6 Binding Update List

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Mobile IPv6, Binding Update List, MIPv6, State Management, RFC 6275

Description: Understand the Mobile Node's Binding Update List, which tracks all outstanding Binding Updates sent to Home Agents and Correspondent Nodes.

## Introduction

The Binding Update List (BUL) is maintained by the Mobile Node and is the counterpart to the Binding Cache maintained by the Home Agent and Correspondent Nodes. The BUL tracks all BUs the MN has sent, their status, and when they need to be refreshed.

## Binding Update List Structure

Each entry in the BUL corresponds to a BU sent to one destination (HA or CN).

```
Binding Update List Entry:
├── Destination Address       — HA or CN address the BU was sent to
├── Home Address (HoA)        — MN's HoA included in the BU
├── Care-of Address (CoA)     — CoA registered in the BU
├── Sequence Number           — BU sequence number sent
├── Lifetime                  — Granted lifetime from BA
├── Refresh Time              — When to send the next BU (lifetime/2)
├── Initial BU Timeout        — For retransmission if no BA received
├── Acknowledged              — True if BA was received
└── Flags
    ├── H: Home Registration  — BU sent to HA
    ├── A: Ack Requested
    └── K: Key Management
```

## BUL in Python (Simplified Implementation)

```python
import time
import threading
from dataclasses import dataclass, field
from typing import Dict, Optional, Callable

@dataclass
class BULEntry:
    destination: str              # HA or CN address
    hoa: str                      # Home Address
    coa: str                      # Care-of Address
    sequence: int                 # BU sequence number
    lifetime: int                 # Granted lifetime (seconds)
    is_home_registration: bool
    acknowledged: bool = False
    created_at: float = field(default_factory=time.time)

    @property
    def refresh_time(self) -> float:
        """Refresh at half the granted lifetime."""
        return self.created_at + (self.lifetime * 0.5)

    @property
    def expiry_time(self) -> float:
        return self.created_at + self.lifetime

    @property
    def needs_refresh(self) -> bool:
        return time.time() >= self.refresh_time and not self.is_expired

    @property
    def is_expired(self) -> bool:
        return time.time() >= self.expiry_time


class BindingUpdateList:
    def __init__(self, send_bu_callback: Callable):
        self._entries: Dict[str, BULEntry] = {}
        self._send_bu = send_bu_callback
        self._seq_counter = 0
        self._lock = threading.Lock()

    def add_or_update(self, destination: str, hoa: str, coa: str,
                      lifetime: int, is_home_reg: bool = True):
        """Add a new BUL entry after sending a BU."""
        with self._lock:
            self._seq_counter = (self._seq_counter + 1) % 65536
            entry = BULEntry(
                destination=destination,
                hoa=hoa,
                coa=coa,
                sequence=self._seq_counter,
                lifetime=lifetime,
                is_home_registration=is_home_reg
            )
            self._entries[destination] = entry
            print(f"BUL: added entry for dst={destination} seq={entry.sequence}")

    def mark_acknowledged(self, destination: str, granted_lifetime: int):
        """Called when a BA is received."""
        with self._lock:
            entry = self._entries.get(destination)
            if entry:
                entry.acknowledged = True
                entry.lifetime = granted_lifetime
                entry.created_at = time.time()  # Reset timer from BA receipt
                print(f"BUL: acknowledged from {destination}, "
                      f"lifetime={granted_lifetime}s")

    def refresh_due_entries(self):
        """Send BUs for entries that need refresh."""
        with self._lock:
            for dst, entry in list(self._entries.items()):
                if entry.needs_refresh:
                    print(f"BUL: refreshing binding with {dst}")
                    self._send_bu(
                        destination=dst,
                        hoa=entry.hoa,
                        coa=entry.coa,
                        lifetime=entry.lifetime
                    )
                elif entry.is_expired:
                    print(f"BUL: entry for {dst} expired, removing")
                    del self._entries[dst]

    def show(self):
        """Print all BUL entries."""
        print("\n=== Binding Update List ===")
        for dst, entry in self._entries.items():
            status = "ACK" if entry.acknowledged else "PENDING"
            print(f"Dst: {dst}, HoA: {entry.hoa}, CoA: {entry.coa}, "
                  f"Seq: {entry.sequence}, TTL: {int(entry.expiry_time - time.time())}s, "
                  f"Status: {status}")
```

## BU Retransmission Logic

If no BA is received, the MN retransmits with exponential backoff.

```python
def send_bu_with_retry(destination, bu_params, max_retries=5):
    """
    Send a BU with exponential backoff retransmission.
    Initial timeout = 1 second, doubles each attempt.
    """
    timeout = 1.0  # Initial retransmission timeout (seconds)

    for attempt in range(max_retries):
        send_bu(destination, **bu_params)

        # Wait for BA
        ba = wait_for_binding_ack(destination, timeout=timeout)
        if ba:
            return ba

        # Exponential backoff
        timeout = min(timeout * 2, 32)  # Max 32 seconds
        print(f"BU retry {attempt + 1}/{max_retries}, "
              f"next timeout={timeout}s")

    raise Exception(f"BU to {destination} failed after {max_retries} retries")
```

## BUL During Handover (CoA Change)

```python
def handle_movement(mn_state, new_coa: str):
    """
    Update BUL when MN moves to a new network.
    """
    old_coa = mn_state.current_coa
    mn_state.current_coa = new_coa

    # Must send BU to ALL entries in the BUL
    for destination, entry in mn_state.bul.entries.items():
        print(f"Updating binding with {destination}: "
              f"old CoA={old_coa} → new CoA={new_coa}")
        mn_state.bul.add_or_update(
            destination=destination,
            hoa=entry.hoa,
            coa=new_coa,  # Updated CoA
            lifetime=entry.lifetime,
            is_home_reg=entry.is_home_registration
        )
```

## Conclusion

The Binding Update List is the Mobile Node's record of all active bindings. Proper BUL management — including timely refresh before expiry and updating all entries on CoA change — is essential for uninterrupted connectivity. Monitor BU success rates and refresh timing issues with OneUptime to detect mobility signaling problems.
