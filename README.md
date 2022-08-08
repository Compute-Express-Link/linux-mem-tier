# linux-mem-tier

```
Transparent Page Placement for Tiered-Memory

From:	 	Hasan Al Maruf <hasan3050-AT-gmail.com>
To:	 	dave.hansen-AT-linux.intel.com, ying.huang-AT-intel.com, yang.shi-AT-linux.alibaba.com, mgorman-AT-techsingularity.net, riel-AT-surriel.com, hannes-AT-cmpxchg.org
Subject:	 	[PATCH 0/5] Transparent Page Placement for Tiered-Memory
Date:	 	Wed, 24 Nov 2021 13:58:25 -0500
Message-ID:	 	<cover.1637778851.git.hasanalmaruf@fb.com>
Cc:	 	linux-mm-AT-kvack.org, linux-kernel-AT-vger.kernel.org
Archive-link:	 	Article
[resend in proper format]

With the advent of new memory types and technologies, we can see different
types of memory together, e.g. DRAM, PMEM, CXL-enabled memory, etc. In
recent future, we can see CXL-Memory be available in the physical address-
space as a CPU-less NUMA node along with the native DDR memory channels.
As different types of memory have different level of performance impact,
how we manage pages across the NUMA nodes should be a matter of concern.

Dave Hansen's patchset on "Migrate Pages in lieu of discard" demotes
toptier pages to a slow tier node during the reclamation process.

    		https://lwn.net/Articles/860215/

However, that patchset does not include the features to promote pages on
slow tier memory node to the toptier one. As a result, pages demoted or
newly allocated on the slow tier node, experiences NUMA latency and hurt
application performance. In this patch set, we augment existing AutoNUMA
mechanism to promote pages from slow tier nodes to toptier nodes.

We decouple reclamation and allocation logics for the toptier node so that
reclamation gets triggered at a higher watermark and demotes colder pages
to the slow-tier memory. As a result, toptier nodes can maintain some free
space to accept both new allocation and promotion from slowtier nodes.
During promotion, we add hysteresis to page and only promote pages that
are less likely to be demoted within a short period of time. This reduces
the chance for a page being ping-ponged across the NUMA nodes due to
frequent demotion and promotion within a short period of time.

We tested this patchset on systems with CXL-enabled DRAM and PMEM tiers.
We find this patchset can bring hotter pages to the toptier node while
moving the colder pages to the slow-tier nodes for a good range of Meta
production workloads with live traffic. As a result, toptier nodes serve
more hot pages and the application performance improves.

Case Study of a Meta cache application with two NUMA nodes
==========================================================
Toptier node: DRAM directly attached to the CPU
Slowtier node: DRAM attached through CXL

Toptier vs Slowtier memory capacity ratio is 1:4

With default page placement policy, file caches fills up the toptier node
and anons get trapped in the slowtier node. Only 14% of the total anons
reside in toptier node. Remote NUMA read bandwidth is 80%. Throughput
regression is 18% compared to all memory being served from toptier node.

This patchset brings 80% of the anons to the toptier node. Anons on the
slowtier memory is mostly cold anons. As the toptier node can not host all
the hot memory, some hot files still remain on the slowtier node. Even
though, remote NUMA read bandwidth reduces from 80% to 40%. With this
patchset, throughput regression is only 5% compared to the baseline of
toptier node serving the whole working set.

Hasan Al Maruf (5):
  Promotion and demotion related statistics
  NUMA balancing for tiered-memory system
  Decouple reclaim and allocation for toptier nodes
  Reclaim to satisfy WMARK_DEMOTE on toptier nodes
  active LRU-based promotion to avoid ping-pong
```
