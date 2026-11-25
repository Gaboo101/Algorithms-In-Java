# Simple Java Snippets for Scheduling Algorithms

Each algorithm is a separate method. Put the methods and helpers inside one Java class, and add the imports below. The CPU examples print the process that runs during each time unit. The disk examples return the head's path and total movement.

```java
import java.util.*;
```

## Shared CPU setup

Use this process class for the CPU snippets. `queue` is used by Multi-Level Queue; MLFQ starts every process in queue 0.

```java
static class Process {
    String id;
    int arrival;
    int burst;
    int remaining;
    int priority;
    int queue;
    int quantumUsed;

    Process(String id, int arrival, int burst, int priority, int queue) {
        this.id = id;
        this.arrival = arrival;
        this.burst = burst;
        this.remaining = burst;
        this.priority = priority;
        this.queue = queue;
        this.quantumUsed = 0;
    }
}

static void resetProcesses(List<Process> processes) {
    for (Process p : processes) {
        p.remaining = p.burst;
        p.quantumUsed = 0;
    }
}

static List<Process> byArrival(List<Process> processes) {
    List<Process> sorted = new ArrayList<>(processes);
    Collections.sort(sorted, new Comparator<Process>() {
        public int compare(Process a, Process b) {
            return Integer.compare(a.arrival, b.arrival);
        }
    });
    return sorted;
}
```

Create processes like this:

```java
List<Process> processes = new ArrayList<>();
processes.add(new Process("P1", 0, 7, 2, 1));
processes.add(new Process("P2", 2, 4, 1, 0));
processes.add(new Process("P3", 4, 1, 3, 1));
```

Run each algorithm on a fresh list, or reset all processes before running the next one.

## CPU scheduling

### 1. Shortest Remaining Time First (SRTF)

The process with the least remaining time runs. Check again every time unit, so a new shorter process can interrupt it.

```java
static void srtf(List<Process> processes) {
    resetProcesses(processes);
    int time = 0;
    int finished = 0;

    while (finished < processes.size()) {
        Process shortest = null;

        for (Process p : processes) {
            if (p.arrival <= time && p.remaining > 0) {
                if (shortest == null || p.remaining < shortest.remaining) {
                    shortest = p;
                }
            }
        }

        if (shortest == null) {
            System.out.println("Idle");
            time++;
            continue;
        }

        System.out.println(shortest.id);
        shortest.remaining--;
        time++;

        if (shortest.remaining == 0) {
            finished++;
        }
    }
}
```

### 2. Preemptive Priority

The process with the smallest priority number runs. Check again every time unit, so a newly arrived higher-priority process can interrupt it.

```java
static void preemptivePriority(List<Process> processes) {
    resetProcesses(processes);
    int time = 0;
    int finished = 0;

    while (finished < processes.size()) {
        Process highest = null;

        for (Process p : processes) {
            if (p.arrival <= time && p.remaining > 0) {
                if (highest == null || p.priority < highest.priority) {
                    highest = p;
                }
            }
        }

        if (highest == null) {
            System.out.println("Idle");
            time++;
            continue;
        }

        System.out.println(highest.id);
        highest.remaining--;
        time++;

        if (highest.remaining == 0) {
            finished++;
        }
    }
}
```

### 3. Round Robin

Each process gets up to `quantum` time units, then goes to the back of the ready queue if it still has work.

```java
static void roundRobin(List<Process> processes, int quantum) {
    if (quantum <= 0) {
        throw new IllegalArgumentException("Quantum must be positive");
    }
    resetProcesses(processes);
    List<Process> arrivals = byArrival(processes);
    Queue<Process> ready = new LinkedList<>();
    int next = 0;
    int time = 0;
    int finished = 0;

    while (finished < processes.size()) {
        while (next < arrivals.size() && arrivals.get(next).arrival <= time) {
            ready.add(arrivals.get(next));
            next++;
        }

        if (ready.isEmpty()) {
            System.out.println("Idle");
            time++;
            continue;
        }

        Process current = ready.remove();
        int runTime = Math.min(quantum, current.remaining);

        for (int i = 0; i < runTime; i++) {
            System.out.println(current.id);
            current.remaining--;
            time++;

            while (next < arrivals.size() && arrivals.get(next).arrival <= time) {
                ready.add(arrivals.get(next));
                next++;
            }
        }

        if (current.remaining == 0) {
            finished++;
        } else {
            ready.add(current);
        }
    }
}
```

### 4. Multi-Level Feedback Queue (MLFQ)

Every process starts in queue 0. If it uses its whole quantum, it moves down one queue. Higher-numbered queues have lower priority. This simple version does not move processes back up.

```java
static int firstNonEmpty(List<Deque<Process>> queues) {
    for (int i = 0; i < queues.size(); i++) {
        if (!queues.get(i).isEmpty()) {
            return i;
        }
    }
    return -1;
}

static boolean hasHigherQueue(List<Deque<Process>> queues, int queueNumber) {
    for (int i = 0; i < queueNumber; i++) {
        if (!queues.get(i).isEmpty()) {
            return true;
        }
    }
    return false;
}

static void mlfq(List<Process> processes, int[] quantum) {
    if (quantum.length == 0) {
        throw new IllegalArgumentException("At least one queue is required");
    }
    for (int q : quantum) {
        if (q <= 0) {
            throw new IllegalArgumentException("Quantum must be positive");
        }
    }
    resetProcesses(processes);
    List<Process> arrivals = byArrival(processes);
    List<Deque<Process>> queues = new ArrayList<>();

    for (int i = 0; i < quantum.length; i++) {
        queues.add(new ArrayDeque<>());
    }

    int next = 0;
    int time = 0;
    int finished = 0;
    Process current = null;

    while (finished < processes.size()) {
        while (next < arrivals.size() && arrivals.get(next).arrival <= time) {
            Process p = arrivals.get(next);
            p.queue = 0;
            queues.get(0).addLast(p);
            next++;
        }

        if (current != null && hasHigherQueue(queues, current.queue)) {
            queues.get(current.queue).addFirst(current);
            current = null;
        }

        if (current == null) {
            int queueNumber = firstNonEmpty(queues);
            if (queueNumber >= 0) {
                current = queues.get(queueNumber).removeFirst();
                current.queue = queueNumber;
            }
        }

        if (current == null) {
            System.out.println("Idle");
            time++;
            continue;
        }

        System.out.println(current.id);
        current.remaining--;
        current.quantumUsed++;
        time++;

        if (current.remaining == 0) {
            finished++;
            current = null;
        } else if (current.quantumUsed == quantum[current.queue]) {
            current.quantumUsed = 0;
            if (current.queue < queues.size() - 1) {
                current.queue++;
            }
            queues.get(current.queue).addLast(current);
            current = null;
        }
    }
}
```

Example quantum setting: `int[] quantum = {2, 4, 8};`

### 5. Multi-Level Queue (MLQ)

Each process stays in its assigned queue. Queue 0 has the highest priority. This example uses Round Robin within each queue.

```java
static void mlq(List<Process> processes, int[] quantum) {
    if (quantum.length == 0) {
        throw new IllegalArgumentException("At least one queue is required");
    }
    for (int q : quantum) {
        if (q <= 0) {
            throw new IllegalArgumentException("Quantum must be positive");
        }
    }
    resetProcesses(processes);
    List<Process> arrivals = byArrival(processes);
    List<Deque<Process>> queues = new ArrayList<>();

    for (int i = 0; i < quantum.length; i++) {
        queues.add(new ArrayDeque<>());
    }

    for (Process p : processes) {
        if (p.queue < 0 || p.queue >= quantum.length) {
            throw new IllegalArgumentException("Invalid process queue");
        }
    }

    int next = 0;
    int time = 0;
    int finished = 0;
    Process current = null;

    while (finished < processes.size()) {
        while (next < arrivals.size() && arrivals.get(next).arrival <= time) {
            Process p = arrivals.get(next);
            queues.get(p.queue).addLast(p);
            next++;
        }

        if (current != null && hasHigherQueue(queues, current.queue)) {
            queues.get(current.queue).addFirst(current);
            current = null;
        }

        if (current == null) {
            int queueNumber = firstNonEmpty(queues);
            if (queueNumber >= 0) {
                current = queues.get(queueNumber).removeFirst();
            }
        }

        if (current == null) {
            System.out.println("Idle");
            time++;
            continue;
        }

        System.out.println(current.id);
        current.remaining--;
        current.quantumUsed++;
        time++;

        if (current.remaining == 0) {
            finished++;
            current = null;
        } else if (current.quantumUsed == quantum[current.queue]) {
            current.quantumUsed = 0;
            queues.get(current.queue).addLast(current);
            current = null;
        }
    }
}
```

## Shared disk setup

`path` stores each head position visited, including disk edges and circular jumps. `movement` is the total number of cylinders traveled.

```java
static class Disk {
    int head;
    int movement = 0;
    List<Integer> path = new ArrayList<>();

    Disk(int start) {
        head = start;
        path.add(start);
    }

    void moveTo(int next) {
        movement += Math.abs(next - head);
        head = next;
        path.add(next);
    }
}

static List<Integer> above(int[] requests, int start) {
    List<Integer> result = new ArrayList<>();
    for (int request : requests) {
        if (request > start) {
            result.add(request);
        }
    }
    return result;
}

static List<Integer> below(int[] requests, int start) {
    List<Integer> result = new ArrayList<>();
    for (int request : requests) {
        if (request < start) {
            result.add(request);
        }
    }
    return result;
}

static void serviceAtStart(Disk disk, int[] requests, int start) {
    for (int request : requests) {
        if (request == start) {
            disk.moveTo(request);
        }
    }
}

static void visit(Disk disk, List<Integer> requests) {
    for (int request : requests) {
        disk.moveTo(request);
    }
}
```

## Disk scheduling

### 6. FCFS

Visit requests in the order they were received.

```java
static Disk fcfs(int[] requests, int start) {
    Disk disk = new Disk(start);
    for (int request : requests) {
        disk.moveTo(request);
    }
    return disk;
}
```

### 7. SSTF (Shortest Seek Time First)

Always visit the pending request closest to the current head.

```java
static Disk sstf(int[] requests, int start) {
    Disk disk = new Disk(start);
    List<Integer> pending = new ArrayList<>();
    for (int request : requests) {
        pending.add(request);
    }

    while (!pending.isEmpty()) {
        int nearestIndex = 0;

        for (int i = 1; i < pending.size(); i++) {
            int distance = Math.abs(pending.get(i) - disk.head);
            int nearestDistance = Math.abs(pending.get(nearestIndex) - disk.head);
            if (distance < nearestDistance) {
                nearestIndex = i;
            }
        }

        disk.moveTo(pending.remove(nearestIndex));
    }
    return disk;
}
```

### 8. SCAN

Move in one direction, go all the way to that disk edge, then reverse.

```java
static Disk scan(int[] requests, int start, int lastCylinder, boolean moveUp) {
    Disk disk = new Disk(start);
    serviceAtStart(disk, requests, start);
    List<Integer> up = above(requests, start);
    List<Integer> down = below(requests, start);

    if (moveUp) {
        Collections.sort(up);
        down.sort(Collections.reverseOrder());
        visit(disk, up);
        if (requests.length > 0) {
            disk.moveTo(lastCylinder);
        }
        visit(disk, down);
    } else {
        down.sort(Collections.reverseOrder());
        Collections.sort(up);
        visit(disk, down);
        if (requests.length > 0) {
            disk.moveTo(0);
        }
        visit(disk, up);
    }

    return disk;
}
```

### 9. C-SCAN (Circular SCAN)

Move in one direction. If requests remain on the other side, go to the edge, jump to the opposite edge, and keep moving in the same direction.

```java
static Disk cScan(int[] requests, int start, int lastCylinder, boolean moveUp) {
    Disk disk = new Disk(start);
    serviceAtStart(disk, requests, start);
    List<Integer> up = above(requests, start);
    List<Integer> down = below(requests, start);

    if (moveUp) {
        Collections.sort(up);
        Collections.sort(down);
        visit(disk, up);
        if (!down.isEmpty()) {
            disk.moveTo(lastCylinder);
            disk.moveTo(0);
            visit(disk, down);
        }
    } else {
        down.sort(Collections.reverseOrder());
        up.sort(Collections.reverseOrder());
        visit(disk, down);
        if (!up.isEmpty()) {
            disk.moveTo(0);
            disk.moveTo(lastCylinder);
            visit(disk, up);
        }
    }

    return disk;
}
```

### 10. LOOK

Like SCAN, but reverse at the last request instead of going to the disk edge.

```java
static Disk look(int[] requests, int start, boolean moveUp) {
    Disk disk = new Disk(start);
    serviceAtStart(disk, requests, start);
    List<Integer> up = above(requests, start);
    List<Integer> down = below(requests, start);

    if (moveUp) {
        Collections.sort(up);
        down.sort(Collections.reverseOrder());
        visit(disk, up);
        visit(disk, down);
    } else {
        down.sort(Collections.reverseOrder());
        Collections.sort(up);
        visit(disk, down);
        visit(disk, up);
    }

    return disk;
}
```

### 11. C-LOOK

Like C-SCAN, but jump between the last request in one direction and the first request on the other side. Do not visit the disk edges.

```java
static Disk cLook(int[] requests, int start, boolean moveUp) {
    Disk disk = new Disk(start);
    serviceAtStart(disk, requests, start);
    List<Integer> up = above(requests, start);
    List<Integer> down = below(requests, start);

    if (moveUp) {
        Collections.sort(up);
        Collections.sort(down);
        visit(disk, up);
        if (!down.isEmpty()) {
            disk.moveTo(down.get(0));
            for (int i = 1; i < down.size(); i++) {
                disk.moveTo(down.get(i));
            }
        }
    } else {
        up.sort(Collections.reverseOrder());
        down.sort(Collections.reverseOrder());
        visit(disk, down);
        if (!up.isEmpty()) {
            disk.moveTo(up.get(0));
            for (int i = 1; i < up.size(); i++) {
                disk.moveTo(up.get(i));
            }
        }
    }

    return disk;
}
```

After running a disk method, inspect its result like this:

```java
Disk result = scan(requests, 53, 199, true);
System.out.println("Head path: " + result.path);
System.out.println("Total movement: " + result.movement);
```
