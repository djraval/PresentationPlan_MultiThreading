# Presentation Plan: Multi-threading and Asynchronous Programming at Morningstar DBRS

# Slide 1
## Introduction
- Discuss the recent issue in our codebase where some requests were timing out
- Explain how I fixed the issue by modifying a list of Class objects to load an attribute via microservice using a thread pool.(Only high level details, Code to follow in slide 3)
<!-- - Don't forget to mention Bijan's suggestion to present this solution to the team -->

# Slide 2

## Multi-threading vs. Asynchronous Programming
### Definitions
- **Multi-threading**: Running multiple threads concurrently within a single process.
- **Asynchronous programming**: Running multiple tasks concurrently without blocking the main thread.

### Key Differences
- **Task switching**: Async code usually runs one block of code at a time and switches to another task (block of code) when it is blocked. This switching is done by the event loop (using `await`). Multi-threading runs multiple threads concurrently and switches between them when a thread is blocked. This switching is done by the operating system (using `yield`).
- **Performance**: Async code is generally faster than multi-threaded code because it does not require context switching between threads. However, it can be more difficult to debug and maintain.
- **Implementation**: Multi-threading is easier to implement with a smaller delta on the codebase and a comparable performance gain. Async approach is best when starting a new project from scratch. For existing projects with large codebases, it might not be worth the performance gain to change the whole code to async.

## Libraries for Multi-threading and Asynchronous Programming
### Standard Libraries
- **Multi-threading**: The `threading` and `concurrent.futures` libraries are standard Python libraries that provide support for multi-threading.
- **Asynchronous programming**: The `asyncio` library is a standard Python library that provides support for asynchronous programming using coroutines.

### Third-party Libraries
- **Dask**: Dask is a popular third-party library that provides advanced parallelism for analytics, enabling users to harness the full power of multi-threading and asynchronous programming.
- **AIOHTTP**: AIOHTTP is a third-party library used for making asynchronous HTTP requests. It is often used as an alternative to the popular `requests` library when working with asynchronous programming. AIOHTTP leverages the `asyncio` library to enable non-blocking I/O operations.

## Benefits and Limitations of Multi-threading and Asynchronous Programming
### Benefits
- **Less idle time for I/O-bound tasks**: Tasks that spend most of their time waiting for external resources, such as reading from a file or fetching data from the internet, can benefit from multi-threading or asynchronous programming.
- **Faster execution for CPU-bound tasks**: Tasks that require heavy computation can be divided into smaller tasks and executed concurrently using multi-threading.
- **Better resource utilization**: By running multiple tasks concurrently, you can make better use of available system resources.

### Limitations
- **Thread safety**: Multi-threading or asynchronous programming should not be used when the code is not thread-safe (e.g., primitive data types, mutable objects).
- **Readability and maintainability**: In cases where the code is only run once, introducing multi-threading or asynchronous programming may complicate the code and make it harder to read and maintain.

### Real-world examples
- Web servers, web scraping, data processing, etc.

# Slide 3
## Code Demonstration
- Show the snippet of the modified code that resolved the timeout issue
``` python
import requests
import time
from concurrent.futures import ThreadPoolExecutor

# Mock IssuerClient class
class IssuerClient:
    def get_isins(self, issuer_id):
        time.sleep(1)
        req_url = f"https://reqres.in/api/users/{issuer_id}"
        response = requests.get(req_url)
        if response.status_code == 200:
            data = response.json()
            print(f'Got isins for issuer {issuer_id}')
            return data['data']['email']
        else:
            raise Exception(f"Request failed with status code {response.status_code}")

#Mock Document class (class where _enrich_issuers() resides)
class Document:
    # Mock issuers list
    issuers =   [
                    {'id':1, 'isins':None},
                    {'id':2, 'isins':None},
                    {'id':3, 'isins':None},
                    {'id':4, 'isins':None},
                    {'id':5, 'isins':None},
                    {'id':6, 'isins':None},
                    {'id':7, 'isins':None},
                    {'id':8, 'isins':None},
                    {'id':9, 'isins':None},
                    {'id':10, 'isins':None},
                    {'id':11, 'isins':None},
                ]
    
    market_sectors  = ['Sovereign']

    def _enrich_single_issuer(self, issuer):
        # Modify as per your actual code
        issuer['isins'] = IssuerClient().get_isins(issuer['id'])
        if not self.market_sectors:
            issuer['type'] = 'Corporate'
        elif self.market_sectors[0] == 'Provinces and Municipalities':
            issuer['type'] = 'Municipality'
        elif self.market_sectors[0] == 'Sovereign':
            issuer['type'] = 'Sovereign'
        else:
            issuer['type'] = 'Corporate'

    def _enrich_issuers(self):
        with ThreadPoolExecutor(max_workers=8) as executor:
            futures = []
            for issuer in self.issuers:
                future = executor.submit(self._enrich_single_issuer, issuer)
                futures.append(future)

            for i, future in enumerate(futures):
                try:
                    future.result()
                except Exception as e:
                    print(f"Exception for issuer {self.issuers[i]['id']}: {str(e)}")

# Driver Code
doc = Document()
print(doc.issuers)
doc._enrich_issuers()
print(doc.issuers)
```
- Walk through the code step-by-step to ensure everyone understands the changes made
- Explain any potential trade-offs or challenges encountered during implementation
    Increased resource usage due to concurrent requests
    Potential rate-limiting by external API if too many requests are sent simultaneously
    Error handling and debugging can be more complex in a concurrent environment

# Slide 4
## Multi-processing 
### Definition of Multi-processing
Multi-processing is a programming technique where multiple processes run concurrently, each having its own memory space and Python interpreter. It can be used over multi-threading in situations where true parallelism is required, particularly for CPU-bound tasks.

### Comparison with Multi-threading
| Multi-processing | Multi-threading |
|------------------|-----------------|
| Multiple processes running concurrently | Multiple threads running within a single process |
| Each process has its own memory space | All threads share the same memory space |
| True parallelism | Concurrency with an illusion of parallelism |
| Better for CPU-bound tasks | Better for I/O-bound tasks |

### The `multiprocessing` Library
The `multiprocessing` library in Python provides an easy-to-use API for creating and managing multiple processes. Here's an example:

```python
import multiprocessing

def worker():
    print("Worker process")

if __name__ == "__main__":
    process = multiprocessing.Process(target=worker)
    process.start()
    process.join()
```

### When to Choose Multi-processing
For CPU-bound tasks, multi-processing is preferred over multi-threading due to Python's Global Interpreter Lock (GIL). The GIL allows only one thread to hold control of the Python interpreter at any given time, limiting the true parallelism potential of multi-threading. In contrast, multi-processing enables true parallelism, as each process runs independently with its own Python interpreter.

# Slide 5

## Best Practices for Multi-threading and Asynchronous Programming
1. **Pass arguments to functions**: Instead of using global variables, pass necessary data as arguments to functions. This helps prevent race conditions and improves code readability.
2. **Use thread-safe data structures**: Opt for thread-safe data structures like `Queue` and `deque` to avoid potential synchronization issues.
3. **Employ ThreadPoolExecutor and ProcessPoolExecutor**: These classes from the `concurrent.futures` module provide a higher-level interface for managing threads or processes, making it easier to create, manage, and monitor them.

## Debugging and Performance Monitoring Tips
1. **Utilize the logging module**: The `logging` module can be used to debug multi-threaded code effectively. Include thread or process IDs in log messages to help identify issues related to specific threads or processes.
2. **Create a single-threaded variant**: Having a single-threaded version of your code can simplify debugging logical issues, as it eliminates concurrency-related complications. Once the logic is correct, you can then reintroduce concurrency to improve performance.
3. **Leverage rich inbuilt debuggers in IDEs**: Integrated Development Environments (IDEs) like PyCharm offer powerful inbuilt debuggers that can be used to step through multi-threaded code, inspect variables, and set breakpoints. These tools can greatly assist in identifying and resolving issues in your code.

# Slide 6
## Conclusion
- Recap of the key points:
    - Multi-threading and asynchronous programming can improve performance for I/O-bound tasks and CPU-bound tasks.
    - There are benefits and limitations to both approaches, and it's important to choose the right one based on the specific requirements of your project.
    - Standard libraries like `threading`, `concurrent.futures`, and `asyncio` provide support for multi-threading and asynchronous programming in Python.
    - Real-world examples include web servers, web scraping, and data processing.
    - Best practices for multi-threading and asynchronous programming include passing arguments to functions, using thread-safe data structures, and employing ThreadPoolExecutor and ProcessPoolExecutor.
    - Debugging and performance monitoring tips include utilizing the logging module, creating a single-threaded variant, and leveraging rich inbuilt debuggers in IDEs.

- Encourage team members to explore and experiment with these concepts in their own projects:
    - Identify any potential areas of improvement in our codebase.
    - Simple formula: If there are no depencies then it could be parallelized.

- Open up the floor for questions and discussion.
