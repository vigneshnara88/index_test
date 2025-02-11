# Experiment: Handling Open-Meteo API Rate Limits

| **Method**             | **Description**                                                                                             | **Rate Limit Handling**                                      | **Drawbacks**                                                                                                                      | **Outcome**                                                                                          |
|-------------------------|-------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------|
| **Batching**           | Divided 1500 coordinates into batches of 50 and sequentially processed them.                               | Tried limiting to 600 requests/min by waiting between batches. | - Took too much time to process all requests.<br>- Inefficient for high volumes due to idle waiting.<br>- Not memory-efficient.   | **Failed**: Took excessive time and was impractical for 1500 requests.                              |
| **ThreadPoolExecutor** | Used 50-100 threads to parallelize requests for each batch.                                                 | No explicit rate throttling per thread.<br>Relied on batching. | - Threads sent requests in bursts, leading to rate limit surges.<br>- Exceeded 600 requests/min after ~6 seconds.<br>- Memory overhead. | **Failed**: Rate limits were exceeded despite batching, causing `429 Too Many Requests` errors.     |
| **Asyncio**            | Used `asyncio` with `aiohttp` for asynchronous requests and semaphores to limit concurrent requests to 10. | Strictly adhered to the rate limit of 10 requests/sec (600/min). | - Requires rewriting code for asynchronous handling.<br>- Steeper learning curve compared to threading.                          | **Success**: Efficiently processed all 1500 requests without exceeding rate limits.                 |

---

### **Key Observations**

| **Observation**                      | **Details**                                                                                                                                         |
|--------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| **Batching Challenges**              | - Batching led to sequential execution with idle waiting, making it too time-intensive.<br>- Not scalable for large datasets like 1500 requests.    |
| **ThreadPoolExecutor Challenges**    | - Threads sent bursts of requests, leading to **rate limit surges**.<br>- Lacked fine-grained control over request timing.<br>- High memory usage.   |
| **Asyncio Advantages**               | - Lightweight and scalable due to single-threaded event loop.<br>- Semaphores ensured strict adherence to rate limits.<br>- Dynamically handled retries. |
| **Rate Limiting with Asyncio**       | - Semaphores controlled concurrency (e.g., 10 requests at a time).<br>- Respected 600 requests/min by limiting to 10 requests/second.                |
| **Overall Outcome**                  | - `asyncio` was the most effective method, balancing compliance with efficiency and scalability.                                                    |

---

### **Conclusion**

Using `asyncio` with proper concurrency controls (e.g., semaphores) is the most effective solution for managing large-scale API requests with strict rate limits:
- **Efficiency**: Processes requests concurrently without exceeding limits.
- **Scalability**: Handles high volumes like 1500 requests seamlessly.
- **Reliability**: Automatically retries rate-limited requests (via `Retry-After`).
