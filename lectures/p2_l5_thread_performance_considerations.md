# Thread Performance Considerations

<img src="which_model_is_better.png">

If we consider execution time, the pipeline threading model is faster. However, if we care about average time to complete order, then the boss-worker model is better. 

<img src="are_threads_useful">

How did we draw these conclusions about threads? How do we determine what is a useful metric? For a matrix multiply application, execution time is probably the metric we want to focus on. For a web service application, it is probably the number of client requests per second. Or maybe response time might be a good metric.

To evaluate a solution, we need to determine the properties that matter, and figure out how to measure that property. 

## Performance Metrics

<img src="performance_metrics.png">

A metric is a measurable quantity that we can use to reason about the behavior of the system. Ideally we can run experiments with real software deployments and real workloads. 