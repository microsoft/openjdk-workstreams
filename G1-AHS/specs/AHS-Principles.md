# Automatic Heap Sizing – Design Principles

_Compiled by Monica Beckwith, May 2025_

---

## 🔑  Guiding Principles
1. **Dynamic feedback loop** – adjust heap on each GC using GC‑CPU %, live‑data, and allocation rate.  
2. **Clear, prioritised signals** –  
   * `SoftMaxHeapSize` (heap‑level soft ceiling)  
   * `GCTimeRatio` (GC‑CPU target; becoming float & runtime manageable)  
   * `G1ReservePercent` (heap‑wide free‑region reserve; safety floor)  
   * `Min/MaxHeapFreeRatio` (legacy Full‑GC guard‑rails only)
3. **Mandatory uncommit** – shrinking must translate into concurrent uncommit (JDK‑8238686) so RSS really falls.
4. **Container awareness, not duplication** – rely on cgroup limits for hard caps; `CurrentMaxHeapSize` therefore stays optional / niche.

---

## 💡  Architectural Control Flow

![G1 AHS Control Flow](../graphs/G1-Control-Flow-2025.png)

---

## ✨  Difference vs ZGC AHS
| Aspect          | ZGC                                  | G1 (planned)                                                      |
|-----------------|--------------------------------------|-------------------------------------------------------------------|
| Feedback signal | GC‑CPU% only                         | GC‑CPU + region occupancy (IHOP feeds back via `SoftMaxHeapSize`) |
| Soft limit      | `SoftMaxHeapSize` rewritten every GC | `SoftMaxHeapSize` guides IHOP; `GCTimeRatio` does resizing        |
| Reserve         | implicit                             | `G1ReservePercent` (heap floor)                                   |
| Uncommit        | fully concurrent                     | concurrent uncommit landing (JDK‑8238686)                         |
| Legacy knobs    | none                                 | Min/MaxHeapFreeRatio kept for Full GC                             |

(See [MPLR 2023 paper](https://dl.acm.org/doi/pdf/10.1145/3617651.3622988) and [Per Liden’s blog](https://malloc.se/blog/zgc-softmaxheapsize) for ZGC details.)

---

## 🔗  Key Issues & References
* JDK‑8238686 (concurrent uncommit), JDK‑8238687 (shrink support)  
* JDK‑8247843 (`GCTimeRatio = 24`), JDK‑8353716 (AHS umbrella)  
* SoftMaxHeapSize PR [#24211](https://github.com/openjdk/jdk/pull/24211)
* [G1ReservePercent adaptive‑reserve discussion – hotspot‑gc‑dev, May 2025](https://mail.openjdk.org/pipermail/hotspot-gc-dev/2025-May/052193.html)
