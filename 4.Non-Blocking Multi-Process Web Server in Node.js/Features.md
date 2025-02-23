# Non-Blocking Multi-Process Web Server in Node.js

## Overview
This project demonstrates the implementation of a **non-blocking, multi-process web server** in **Node.js**. The server efficiently handles computationally expensive tasks, such as checking if a number is prime, by utilizing **child processes** and a **worker pool**. This architecture ensures high performance and scalability by leveraging multiple CPU cores while keeping the main event loop responsive.

## Features
- **Non-blocking request handling**
- **Child process-based computation**
- **Worker pool implementation for parallel processing**
- **Efficient query parameter validation**
- **Timeout and error handling for reliability**

## How It Works
### 1. Handling Expensive Computations Without Blocking
Instead of performing CPU-intensive tasks in the main event loop, the server forks a separate process to handle computations.

```javascript
const http = require('http');
const { fork } = require('child_process');

const server = http.createServer((req, res) => {
  const worker = fork('./worker.js');
  
  worker.send({ input: computationInput });

  worker.on('message', (result) => {
    res.end(JSON.stringify(result));
  });
});
```

### 2. Using Child Processes for Asynchronous Computation
The **fork()** method is used to create a child process that executes computations separately.

#### **Main Server File:**
```javascript
const { fork } = require('child_process');

function computeInChildProcess(input) {
  const worker = fork('./worker.js');
  
  return new Promise((resolve, reject) => {
    worker.send({ input });
    worker.on('message', (result) => {
      resolve(result);
      worker.kill();
    });
    worker.on('error', (err) => {
      reject(err);
      worker.kill();
    });
  });
}
```

#### **Worker File (worker.js):**
```javascript
process.on('message', (msg) => {
  const result = performExpensiveCalculation(msg.input);
  process.send({ result });
});

function performExpensiveCalculation(number) {
  let isPrime = true;
  for (let i = 2; i <= Math.sqrt(number); i++) {
    if (number % i === 0) {
      isPrime = false;
      break;
    }
  }
  return { number, isPrime };
}
```

### 3. Query Parameter Validation
A validation function ensures correct query parameters before processing the request.

```javascript
function validateQueryParams(query) {
  if (!query.number) {
    return { valid: false, error: 'Missing required parameter: number' };
  }
  const number = parseInt(query.number, 10);
  if (isNaN(number)) {
    return { valid: false, error: 'Parameter "number" must be an integer' };
  }
  return { valid: true, params: { number } };
}
```

### 4. Worker Pool Implementation
A worker pool efficiently manages worker processes, avoiding unnecessary creation and destruction of processes.

```javascript
class WorkerPool {
  constructor(workerScript, minWorkers = 2, maxWorkers = 8) {
    this.workerScript = workerScript;
    this.maxWorkers = maxWorkers;
    this.workers = [];
    this.taskQueue = [];
    for (let i = 0; i < minWorkers; i++) {
      this.addWorker();
    }
  }
  addWorker() {
    const worker = fork(this.workerScript);
    worker.status = 'idle';
    worker.on('message', (result) => {
      if (worker.currentTask) {
        worker.currentTask.resolve(result);
        worker.currentTask = null;
        worker.status = 'idle';
        this.processQueue();
      }
    });
    this.workers.push(worker);
  }
  executeTask(data) {
    return new Promise((resolve, reject) => {
      this.taskQueue.push({ data, resolve, reject });
      this.processQueue();
    });
  }
  processQueue() {
    if (this.taskQueue.length === 0) return;
    const idleWorker = this.workers.find(worker => worker.status === 'idle');
    if (idleWorker) {
      const task = this.taskQueue.shift();
      idleWorker.currentTask = task;
      idleWorker.status = 'busy';
      idleWorker.send(task.data);
    }
  }
}
```

### 5. Timeout and Error Handling
Timeouts prevent long-running tasks from hanging the system.

```javascript
function executeWithTimeout(workerPool, task, timeout) {
  return new Promise((resolve, reject) => {
    const timeoutId = setTimeout(() => {
      reject(new Error(`Task timed out after ${timeout}ms`));
    }, timeout);
    workerPool.executeTask(task)
      .then(result => {
        clearTimeout(timeoutId);
        resolve(result);
      })
      .catch(err => {
        clearTimeout(timeoutId);
        reject(err);
      });
  });
}
```

## Setup & Installation
### **Prerequisites**
- Node.js installed (version 14+ recommended)
- Basic understanding of JavaScript and Node.js

### **Installation Steps**
1. Clone the repository:
   ```sh
   git clone https://github.com/your-repo/non-blocking-server.git
   cd non-blocking-server
   ```
2. Install dependencies:
   ```sh
   npm install
   ```
3. Start the server:
   ```sh
   node server.js
   ```

## API Usage
### **Checking if a Number is Prime**
#### **Request:**
```sh
GET http://localhost:8080/?number=23
```
#### **Response:**
```json
{
  "number": 23,
  "isPrime": true
}
```

## Benefits of This Architecture
✅ **Non-Blocking:** The server can handle multiple requests efficiently.
✅ **Parallel Processing:** Utilizes multiple CPU cores for performance.
✅ **Scalability:** Worker pool manages tasks dynamically.
✅ **Fault Tolerance:** Handles errors and timeouts gracefully.
✅ **Efficient Resource Utilization:** Prevents main event loop from being blocked.