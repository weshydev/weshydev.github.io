---
layout: post
title: Building a Load-Shedding System in Go. Prioritizing Tasks Under Heavy Load
date: 2024-11-25 12:00:00
description: Learn how to build a load-shedding system in Go to prioritize critical tasks under heavy load. This guide simulates real-world scenarios, like handling video playback and analytics, using task queues, priority-based logic, and worker pools to keep systems responsive. Perfect for exploring backend resilience techniques!
tags: distributed-systems
categories: backend
thumbnail: assets/img/load_shedding_thumbnail.png
---

Have you ever wondered how big tech companies keep their services running smoothly, even when millions of users are online at the same time? One of the secrets behind this magic is a concept called **load shedding**. In this blog post, we'll dive into what load shedding is and build a small-scale system in Go to see it in action.

## **Real-Life Scenario: Video Streaming Platform**

Imagine you're running a popular video streaming platform. Users request video playback, subtitles, recommendations, and analytics. But when traffic become heavy say, during the release of a new movie, your servers can’t handle everything at once. Here’s what happens:

1. **Critical Task**: Delivering the video playback stream is non-negotiable.
2. **Important Task**: Subtitles and audio syncing enhance user experience but aren't as critical as playback.
3. **Optional Task**: Collecting analytics for recommendations is great but can be dropped without affecting users.

**Load shedding** ensures your system prioritizes the playback stream and subtitles while shedding analytics during heavy traffic.

## **The Plan**

We’ll create:

1. **Task Generator**: Simulates user requests (e.g., playback, subtitles, analytics).
2. **Load Manager**: Decides which tasks to accept or drop based on system load.
3. **Worker Pool**: Processes tasks concurrently.
4. **Logger**: Tracks accepted, processed, and dropped tasks.

## **Step 1: Task Structure**

First, define the structure of a task in Go:
```go
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

type Task struct {
	ID       int
	Priority string
	Duration time.Duration
}
```

Each task represents a user request. For example:

- **Playback**: High priority
- **Subtitles**: Medium priority
- **Analytics**: Low priority

## **Step 2: Load Manager**

The Load Manager controls the system's task queue and ensures only tasks within capacity are processed.
```go
type LoadManager struct {
	mu          sync.Mutex
	taskQueue   []Task
	maxCapacity int
}

func NewLoadManager(capacity int) *LoadManager {
	return &LoadManager{
		taskQueue:   make([]Task, 0),
		maxCapacity: capacity,
	}
}
```

**Accepting Tasks**

Tasks are accepted or dropped based on priority:
```go
func (lm *LoadManager) AcceptTask(task Task) bool {
	lm.mu.Lock()
	defer lm.mu.Unlock()

	if len(lm.taskQueue) >= lm.maxCapacity {
		if task.Priority == "Low" {
			fmt.Printf("Task %d dropped (Analytics - Low priority)\n", task.ID)
			return false
		}
		lm.shedLoad(task.Priority)
	}

	lm.taskQueue = append(lm.taskQueue, task)
	fmt.Printf("Task %d accepted (Priority: %s)\n", task.ID, task.Priority)
	return true
}
```

**Shedding Load**

When capacity is full, the system drops lower-priority tasks to make space:
```go
func (lm *LoadManager) shedLoad(minPriority string) {
	priorities := map[string]int{"High": 1, "Medium": 2, "Low": 3}

	newQueue := make([]Task, 0)
	for _, task := range lm.taskQueue {
		if priorities[task.Priority] < priorities[minPriority] {
			newQueue = append(newQueue, task)
		} else {
			fmt.Printf("Task %d shed (Priority: %s)\n", task.ID, task.Priority)
		}
	}
	lm.taskQueue = newQueue
}
```

**Fetching Tasks for Processing**

Workers will use this function to fetch tasks:
```go
func (lm *LoadManager) GetNextTask() *Task {
	lm.mu.Lock()
	defer lm.mu.Unlock()

	if len(lm.taskQueue) == 0 {
		return nil
	}

	task := lm.taskQueue[0]
	lm.taskQueue = lm.taskQueue[1:]
	return &task
}
```

## **Step 3: Worker Pool**

Workers process accepted tasks concurrently:
```go
func worker(id int, lm *LoadManager, wg *sync.WaitGroup) {
	defer wg.Done()

	for {
		task := lm.GetNextTask()
		if task == nil {
			time.Sleep(100 * time.Millisecond)
			continue
		}

		fmt.Printf("Worker %d processing task %d (Priority: %s)\n", id, task.ID, task.Priority)
		time.Sleep(task.Duration)
	}
}
```

## **Step 4: Task Generator**

Generate user requests with random priorities and durations:
```go
func generateTask(id int) Task {
	priorities := []string{"High", "Medium", "Low"}
	duration := time.Duration(rand.Intn(2000)+500) * time.Millisecond

	return Task{
		ID:       id,
		Priority: priorities[rand.Intn(len(priorities))],
		Duration: duration,
	}
}

```

## **Step 5: Main Function**

Combine everything to simulate the load-shedding system:
```go
func main() {
	rand.Seed(time.Now().UnixNano())

	maxCapacity := 10
	numWorkers := 3
	totalTasks := 50

	loadManager := NewLoadManager(maxCapacity)
	var wg sync.WaitGroup

	for i := 1; i <= numWorkers; i++ {
		wg.Add(1)
		go worker(i, loadManager, &wg)
	}

	for id := 1; id <= totalTasks; id++ {
		task := generateTask(id)
		loadManager.AcceptTask(task)
		time.Sleep(time.Duration(rand.Intn(500)) * time.Millisecond)
	}
	
	wg.Wait()
}
```

## **Step 6: Running the Program**

Save your code in `main.go` and run it:
```bash
go run main.go
```

## **What You’ll See**

The logs will show something like this:
```bash
Task 1 accepted (Priority: Medium)
Task 2 accepted (Priority: Low)
Task 3 accepted (Priority: High)
Worker 1 processing task 1 (Priority: Medium)
Worker 2 processing task 3 (Priority: High)
Task 15 dropped (Low priority)
Task 18 shed (Priority: Low)
```

**Key Observations**:

- High-priority tasks (e.g., playback) are always accepted.
- Medium-priority tasks (e.g., subtitles) are processed when capacity allows.
- Low-priority tasks (e.g., analytics) are the first to be dropped under heavy load.

## Conclusion

This system mirrors real-world scenarios like video streaming platforms prioritizing playback and subtitles over analytics. Understanding and implementing load shedding is essential for building resilient systems that can handle high traffic without buckling under pressure.

You can find the complete code on [Github](https://github.com/omarelweshy/golang-load-shedding)
