# number of connected components in an undirected graph: two clean solutions in Python, the Union-Find counting trick, and the edge cases that break most attempts

You're given `n` nodes and a list of undirected edges. Return how many separate groups those nodes form. That's the whole problem — and it's the kind of task that looks trivial until an interviewer asks a follow-up, or until one of your test cases returns 0 because every node in the input happens to be isolated.

This is LeetCode 323, and it holds a slightly odd position in the interview prep world: it's a graph problem that people rarely grind directly, but it shows up constantly as a building block inside harder ones. Kruskal's algorithm, cycle detection, "are these two things in the same group," accounts merge, graph valid tree — they all sit on top of the same primitive. Get comfortable here and five other problems get easier.

Below are the two solutions that matter, when each one is the right answer, and where the pattern sits in a structured curriculum if you'd rather learn the technique than memorise one problem.

## What the problem is actually asking

A connected component is a set of nodes where you can walk from any node in the set to any other using the given edges — and where you can't walk out to a node outside the set. A graph with three isolated clusters has three components. A single node with no edges is a component all by itself. So "no edges" doesn't mean "zero components"; it means `n` components.

The two examples that get used everywhere:


n = 5, edges = [[0,1], [1,2], [3,4]]   →  2


Nodes 0, 1, 2 form one cluster and 3, 4 form another. Now connect the tail:


n = 5, edges = [[0,1], [1,2], [2,3], [3,4]]   →  1


Constraints, per the standard version of the problem: `n` up to 2000, edges up to 5000, no duplicate edges, no self-loops (`a != b`). Those numbers matter for one reason — they're small enough that almost anything correct passes, which means interviewers judge you on the approach you chose, not on whether you squeezed out a constant factor.

## The two approaches, side by side

|  | DFS / BFS | Union-Find (DSU) |
| --- | --- | --- |
| Time | `O(V + E)` | `O(V + E·α(V))`, effectively linear |
| Space | `O(V + E)` for the adjacency list | `O(V)` for parent/size arrays |
| Code length | ~15 lines | ~25 lines |
| Handles edges arriving one at a time | No — you'd re-run the traversal | Yes, `O(α(V))` per edge |
| What it really teaches | Graph traversal | Set merging |

`α(V)` is the inverse Ackermann function, and for any input you'll ever see it's below 5 — Dijkstra's own analysis pegs it as practically constant. So the two are asymptotically the same for a one-shot query. The tiebreaker is the follow-up question, not the complexity.

## DFS: the one you can write in four minutes

Build an adjacency list, walk every unvisited node, count how many times you had to start a walk.

python
def countComponents(n, edges):
    adj = [[] for _ in range(n)]
    for u, v in edges:
        adj[u].append(v)
        adj[v].append(u)   # both directions — this is the line people forget

    visited = [False] * n
    components = 0

    def dfs(start):
        stack = [start]
        while stack:
            cur = stack.pop()
            for nei in adj[cur]:
                if not visited[nei]:
                    visited[nei] = True   # mark when discovered, not when popped
                    stack.append(nei)

    for node in range(n):
        if not visited[node]:
            visited[node] = True
            dfs(node)
            components += 1

    return components


Two details worth saying out loud in an interview. First, mark a node visited the moment you push it, not when you pop it — otherwise the same node lands on the stack several times through different neighbours. Second, the outer loop runs `range(n)`, not `range(len(edges))`. Nodes that appear in no edge still exist.

Swap the stack for a `deque` and the same skeleton becomes BFS. Beyond traversal order, nothing changes.

## Union-Find: fewer moving parts than people expect

Union-Find starts from the opposite assumption. Instead of exploring a graph, it assumes every node is its own component — `n` of them — and then asks a simpler question for each edge: were these two nodes already in the same group? If yes, nothing changes. If no, two groups become one, and the count drops by exactly one.

That single sentence is the whole algorithm:

python
class DSU:
    def __init__(self, n):
        self.parent = list(range(n))
        self.size = [1] * n
        self.components = n

    def find(self, x):
        while x != self.parent[x]:
            self.parent[x] = self.parent[self.parent[x]]  # path halving
            x = self.parent[x]
        return x

    def union(self, a, b):
        ra, rb = self.find(a), self.find(b)
        if ra == rb:
            return                      # already connected, count stays put
        if self.size[ra] < self.size[rb]:
            ra, rb = rb, ra
        self.parent[rb] = ra            # attach the smaller tree under the bigger one
        self.size[ra] += self.size[rb]
        self.components -= 1            # exactly one successful merge, one fewer group

def countComponents(n, edges):
    dsu = DSU(n)
    for u, v in edges:
        dsu.union(u, v)
    return dsu.components


Two optimisations are doing quiet work here. Path halving flattens the tree on every `find` so the next lookup is cheaper. Union by size makes sure you're hanging a shallow tree off a deep one, never the reverse. Together they're what keeps each operation effectively constant instead of degrading toward `O(n)` on a badly shaped chain of parents.

There's also a lazier variant that skips the counter and counts distinct roots at the end:

python
return len({dsu.find(i) for i in range(n)})


Correct, and `O(n·α(n))`, which is fine at these constraints. But if the problem statement says component counts are reported *as edges get added*, the counter version is the one you want, and saying why — "one merge, one decrement, so I can answer online without recomputing" — is the answer an interviewer is fishing for.

## Three mistakes that actually cost people this problem

**Counting from the edge list.** Looping over `edges` and counting connected pairs misses every node that appears in no edge. `n = 5, edges = []` should return 5. Return 0 and the interviewer knows you read the examples and skipped the constraints.

**Building a directed graph.** `adj[u].append(v)` without `adj[v].append(u)` produces a traversal that walks downhill and stops. The counts come out too high, and it can be hard to see why on a small hand-written test case.

**Decrementing on already-connected pairs.** In Union-Find, an edge whose endpoints share a root means the graph has a cycle — nothing merged, so nothing decrements. Decrementing anyway undercounts components, and cycles are exactly where the bug hides.

A fourth, less structural one: treating the recursion depth as harmless. With `n` up to 2000 a recursive DFS survives fine in Python by default, but the iterative stack version removes the question entirely and takes the same effort to write.

## Knowing which variant you're being tested on

This problem is a shape, not a single question. The same machinery answers:

- **Number of Provinces (547)** — the same count, dressed in friendship matrices.
- **Graph Valid Tree (261)** — exactly `n-1` edges *and* exactly one component.
- **Redundant Connection (684)** — the first edge whose two endpoints already share a root.
- **Accounts Merge (721)** — union by shared email, then group.
- **Longest Consecutive Sequence (128)** — union consecutive integers, count components.
- **Kruskal's MST** — sort edges by weight, union greedily.

Notice the pattern: in every case you're asking "are these two things in the same group, and did that just change?" That question is what Union-Find is for. DFX and BFS answer "what's reachable from here," which is a different question that happens to produce the same number on the static version.

A decision rule that survives interviews: static graph, one count, no follow-ups — DFS or BFS, because you'll write it faster. Streaming edges, repeated connectivity queries, or a follow-up about merging — Union-Find, because re-running a traversal for every query is the answer they're hoping you won't give.

## Where the pattern actually lives in a curriculum

Solving one graph problem teaches you that problem. The reason people stall on the follow-ups is that they learned 323 in isolation instead of learning set merging as a tool with a signature, a template, and a list of problems that share it.

That's the gap AlgoMonster's format is built around. It's a text-and-diagram curriculum, not a video library, organised by pattern rather than by problem number. This exact problem sits in the [Disjoint Set Union / Union Find module](https://algo.monster/problems/number_of_connected_components/?fpr=li69), under Advanced Data Structures — and the version there is deliberately framed around connecting nodes and reporting the component count as edges arrive, which is the online formulation that makes Union-Find's advantage concrete rather than theoretical. The DSU fundamentals lesson covers quick-find versus quick-union, rank, and path compression, and its pages carry community discussion threads where implementation details like path halving get picked apart.

If you want to see how the pattern framing reads before paying anything: 👉 [read AlgoMonster's walkthrough of Number of Connected Components](https://algo.monster/problems/number_of_connected_components/?fpr=li69).

Content is written by engineers from Google and competitive programming backgrounds, and the paid tier unlocks every pattern, lesson and problem, unlimited use of the built-in AI assistant, unlimited Speedrun (timed practice), and 3,000+ editorials. Subscribers also get the add-on courses — a CS fundamentals course and a build-an-LLM-from-scratch course — included rather than sold separately.

## What it costs, and which plan makes sense here

AlgoMonster has no monthly option. It's yearly or one-time, plus a free tier with a limited subset of content.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems — enough to judge the teaching style | $0 | — | [Start free](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems, unlimited AI assistant, unlimited Speedrun, 3,000+ editorials, company questions updated monthly, member discount on live bootcamps | $99/year (list price $198) | Billed annually | [Get the yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan, plus lifetime updates and an internal referral to a top tech company for qualified candidates | $189 one-time (list price $378) | Billed once | [Get lifetime access](https://algo.monster/subscribe?fpr=li69) |

Two honest caveats. AlgoMonster's promotional pricing moves — the subscribe page is the only number that matters on the day you buy, and the discounts shown there are usually applied already. And the lifetime tier only pays off if you'll interview more than once. If you're prepping for a single loop over the next two months, the yearly plan covers it; if you expect to change jobs again in three years, one payment beats two renewals.

The scope limits are worth knowing before you buy. There are no video walkthroughs, the problem bank is in the hundreds rather than thousands, and system design and behavioural content is introductory. If you need senior-level system design or mock interviews, that's a separate purchase somewhere else — factor it in.

## Questions people ask before they start

**Do I need Union-Find to pass this problem?** No. DFS or BFS at `O(V + E)` is fully accepted, and faster to type under pressure. Union-Find matters because of the follow-ups and because it unlocks a family of problems.

**Which should I say first in an interview?** Offer both, then commit. "Traversal gives me `O(V + E)`; Union-Find gives me the same practically and handles streaming edges — I'll write the traversal first unless you want the incremental version." That sentence covers three minutes of silence.

**Is the recursive DFS safe at `n = 2000`?** In Python, yes, comfortably. In languages with smaller default stacks and much larger `n`, no. Iterative traversal avoids the whole conversation.

**How long should this take?** If you can write correct DFS or DSU code for it in under ten minutes without checking syntax, you've internalised the pattern. If you're re-deriving path compression each time, that's the gap worth closing — it's the same fifteen lines in a dozen problems, and it's the kind of thing a structured pattern course drills faster than problem-hopping does.

The larger point: the number of connected components in an undirected graph is a small problem with an unusually large shadow. Learn the count, learn the merge, learn when the count has to survive new edges arriving — and you've picked up a tool you'll reuse for years.
