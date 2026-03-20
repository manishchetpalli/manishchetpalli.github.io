## **Spark Streaming**

>--- **Converting Batch to Streaming Code**

One of the core benefits of Spark is that converting batch code to streaming is straightforward.

1. Reading: Change `.read` to `.readStream`.
2. Source: Change the format from `text` to `socket` and provide `host` and `port` options.
3. Transformations: The logic for `split`, `explode`, and `groupBy` remains exactly the same.
4. Writing: Instead of `.show()`, use `.writeStream` with a defined `format` (e.g., `console`) and an `outputMode`.
5. Driver Connection: Add `.awaitTermination()` to ensure the driver stays active while the streaming application runs on executors.

>--- **Key Streaming Concepts**

- Output Mode ("complete"): In this mode, the entire updated result table is displayed in the console every time a new batch is processed.

- Micro-Batches: When you type "hello world" into the terminal, Spark processes it as the first batch. If you type "world" again, it processes a second batch and updates the count for "world" to 2.

- Checkpoints: Spark automatically creates a temporary checkpoint location to store metadata for the streaming application. Checkpoints ensure that the application can track its progress and state.

- Schema Handling: For socket sources, Spark defaults to a schema with a single column called `value` of type string.


>--- **Spark Streaming Output Modes**

Spark offers three primary output modes that define how the result of a transformation is written to the output sink.

- Complete Mode

In Complete Mode, the entire updated result table is written to the sink every time a micro-batch is processed.Even if a specific record was not present in the current micro-batch, it will still appear in the output if it was part of a previous batch.

- Update Mode

In Update Mode, Spark only outputs the rows that were updated or newly added in the most recent micro-batch. If a record from a previous batch is not updated by the new data, it is excluded from the current output.

- Append Mode

In Append Mode, only new rows added to the result table since the last trigger are outputted. Once data is written, it is "locked" and cannot be updated. This makes it ideal for logging where data is only added at the bottom. 

!!! Note
    Append mode is often used in conjunction with watermarks to handle stateful data, which is discussed in more advanced sessions.

So in summary:

-  Complete mode gives you the entire Result Table every time. It's like taking a complete snapshot of your computation's current state after every batch.
- Append mode only gives you completely new rows. It's suitable for when your Result Table is effectively growing over time with new data, such as when you're just adding new rows and not updating existing ones.
-  Update mode gives you any rows that are new or have been updated. It's a middle ground between complete and append mode, giving you a view of what's changed in your Result Table since the last batch.


>--- **Lambda Architecture**

Lambda architecture is defined by having two distinct processing pipelines: a Batch Layer and a Speed (Streaming) Layer.

- How it Works:
    Batch Layer: Processes raw data in large volumes at scheduled intervals (e.g., daily or weekly). This layer is used when high latency is acceptable.
    Speed Layer: Also known as the streaming pipeline, it processes data in real-time for immediate insights.
    Serving Layer: A single, common layer that connects to various applications (real-time, batch, or mixed) and queries the results from both the batch and speed layers.
- Challenges:
    Code Duplication: Because there are two separate pipelines, you often have to write and maintain the same logic twice—once for the batch code and once for the streaming code.
    Latency: Data processed through the batch pipeline is subject to delays because it only runs based on a schedule.

>--- **Kappa Architecture**

Kappa architecture is a simpler alternative that utilizes only one processing pipeline: the Speed Layer.

- How it Works:

    Unified Pipeline: The same streaming pipeline is used to process both real-time data and historical (batch) data.
    Single Flow: All raw data flows through the streaming pipeline and is served through a common data serving layer to all applications.
    Efficiency: It solves the problem of code duplication because you only maintain a single codebase.
- Challenges:

    Out-of-Order Data: Because it is solely based on speed and continuous streaming, data may arrive out of chronological order, which requires handling via "watermarks" (a topic for future discussion).

>--- **Comparison Summary**

| Feature | Lambda Architecture | Kappa Architecture |
| :--- | :--- | :--- |
| Pipelines | Two (Batch + Speed) | One (Speed/Streaming) |
| Complexity | Higher (Code duplication) | Lower (Single codebase) |
| Latency | High in batch layer | Low (Real-time delivery) |
| Data Order | Handled by batch reprocessing | Challenges with out-of-order data |

When setting up the Spark session for streaming, a specific configuration is recommended to ensure data integrity during shutdowns.

- Graceful Shutdown: Setting `spark.streaming.stopGracefullyOnShutdown` to `true` ensures that if the application is shut down, it finishes processing any data already "in line" before stopping the pipeline.
- Schema Inference: For streaming, you must explicitly enable schema inference to allow Spark to identify the JSON structure at runtime.

```python
# Configuration for streaming schema inference and graceful shutdown
spark.conf.set("spark.sql.streaming.schemaInference", "true")
spark.conf.set("spark.streaming.stopGracefullyOnShutdown", "true")
```

>--- **Advanced File Streaming Options**

- `cleanSource`: Controls what happens to the input file after it is read. Options include `off` (default), `delete`, or `archive`.
- `sourceArchiveDir`: Required when using `archive`; it specifies the folder where processed files are moved.
- `maxFilesPerTrigger`: Determines how many files are consumed in a single micro-batch. Setting this to `1` ensures Spark processes only one file at a time, which is useful for controlling resource usage in production.

>--- **Spark Streaming Trigger Types**

Triggers define the timing of streaming data processing. The sources outline three main types:

- Once (or `availableNow`)

This trigger causes the streaming pipeline to behave like a batch job. 
It consumes all data currently available in the source, processes it, and then automatically shuts down the pipeline. Ideal for Kappa architectures where you want to use the same streaming code to process historical data as a one-time batch.

- Processing Time

This is the standard micro-batch trigger where you specify a recurring time interval.
Spark triggers a micro-batch at the defined interval (e.g., every 10 seconds). It processes any new data that arrived during that window.

- Continuous (Experimental)

Currently experimental in Spark 3.3.0, this mode offers the lowest latency.
It does not use micro-batches. Instead, it processes data continuously. The time interval provided (e.g., '10 seconds') defines how often Spark records a checkpoint, not how often it processes data.
It does not yet support all transformations (such as `explode`) or all sinks. For example, it is demonstrated using a memory sink.

>--- **The Problem with Multiple `writeStream` Commands**

A common mistake is trying to call `writeStream` multiple times on the same streaming DataFrame to send data to 
different locations. This approach causes several issues:

Each `writeStream` command triggers its own Spark job for every micro-batch. This means the entire DAG (Directed Acyclic Graph) is re-computed, and the source data is read twice for two different streams.

Each stream must maintain its own checkpoint location to track metadata.Because different sinks have different latencies, one stream might process data faster than the other, leading to them processing different offsets at any given time.

- The Solution: `foreachBatch`

To handle multiple sinks effectively, Spark provides the `foreachBatch` function. This function takes a Python function as input and executes it for every micro-batch.

The data is read from the source and processed through the transformations only once per micro-batch. You only need to maintain one checkpoint location for the entire process. Within the provided Python function, you can use standard batch `write` commands to send data to as many locations as needed.

>--- **Event Time vs. Processing Time**

Understanding the distinction between these two timestamps is vital for accurate streaming analytics.

- Event Time: 
The time at which the data was actually generated at the source (e.g., the moment a sensor in Sydney records a temperature).

- Processing Time: 
The time at which the data arrives at the processing engine (e.g., Spark) to be ingested.

>--- **The Problem of Late Arrival**

Due to geographical distances or network latency, data generated at the same time might arrive at the processing center at different times. 
   Scenario: A device in Delhi (D1) and a device in Sydney (D2) both record a temperature at 12:04 (Event Time).
   Result: D1 arrives almost instantly, while D2 arrives at 1:10 (Processing Time).
   Miscalculation: If you aggregate by processing time, D1 falls into the 12:00–1:00 window, while D2 falls into the 1:00–2:00 window. This leads to incorrect averages because D2 should have been included in the 12:00 window based on its actual generation time.

>--- **Stateful Processing**

To perform aggregations like "hourly average temperature" correctly, Spark must perform stateful processing.

   Logic: Spark holds the "state" (the current sum and count of temperatures) for specific time windows in its memory.
   Updating State: When data generated at 12:04 arrives late (even hours later), Spark must go back into its memory, find the 12:00–1:00 window, and update the calculation.

>--- **Managing Memory with Watermarks**

Spark cannot hold state in its memory indefinitely; otherwise, it would eventually run out of memory (OOM), especially if data arrives days or weeks late.

Watermarks: A watermark is a threshold or "timeout" that tells Spark how long to keep the state for a specific window in memory.

Discarding Late Data: If you define a watermark of 2 hours, Spark will wait for late data for up to two hours after the window closes. Any data arriving after that period is automatically discarded to free up memory.

> --- **Summary Table**

| Concept | Definition | Importance |
| :--- | :--- | :--- |
| Event Time | When data was generated. | Essential for accurate analytics. |
| Processing Time | When data arrived at Spark. | Easier to track but can lead to miscalculations. |
| State | Data kept in Spark's memory. | Allows updates to old windows when late data arrives. |
| Watermark | A "timeout" for late data. | Prevents memory exhaustion by discarding very late data. |

>--- **Overview of Window Operations**

Window operations are essential for stateful processing, allowing you to perform group aggregations (like word counts or temperature averages) over specific time segments. These operations rely on event time to ensure that data is placed into the correct chronological bucket, even if it arrives out of order.

- Tumbling Windows (Fixed Windows)

Tumbling windows are of a fixed, constant size and do not overlap. Once a window ends, a new one begins immediately. Each event belongs to exactly one window.

Example Logic: With a 10-minute window and a 5-minute trigger, Spark processes data in non-overlapping blocks (e.g., 12:00–12:10, 12:10–12:20).

Results for a specific window only get updated when events with an event time falling within that window arrive.


- Sliding Windows (Overlapping Windows)

Sliding windows are also of a fixed size but overlap each other for a specific duration.
Because they overlap, a single event can fall into multiple windows simultaneously.

Example Logic: If the window size is 10 minutes and the "slide" duration is 5 minutes, the windows will overlap by 5 minutes (e.g., W1: 12:00–12:10 and W2: 12:05–12:15).

When an event arrives at 12:07, Spark must update both W1 and W2 because 12:07 falls within both ranges.


- Session Windows

Session windows do not have a fixed size. Instead, they are defined by a session gap or a period of inactivity.
A session stays "open" as long as events are flowing. It automatically terminates once the specified gap duration passes without any new activity.

Example Logic: If the session gap is 5 minutes and the last event occurred at 12:09, the window will terminate at 12:14. If a new user logs in at 12:15 and has no further activity, that session terminates at 12:20.

>--- **The Role of Watermarks in Windowing**

In stateful windowing, Spark must keep aggregations in memory to update them when late data arrives. Watermarks prevent memory exhaustion by defining a threshold for how long Spark should wait for late events.

The watermark is calculated as the `Latest Event Time - Watermark Duration`.

If the latest event is 12:17 and the watermark is 10 minutes, the threshold is 12:07. Any event generated before 12:07 that arrives at 12:20 will be ignored and will not update previous windows. This allows Spark to clear old aggregations from its memory.

- Summary Table

| Window Type | Fixed Size? | Overlapping? | Trigger Basis |
| :--- | :--- | :--- | :--- |
| Tumbling | Yes | No | Fixed time intervals |
| Sliding | Yes | Yes | Fixed time with overlap |
| Session | No | No | Inactivity/Session gap |

>--- **Implementing Windows and Watermarks**

- Tumbling (Fixed) Window: A window of a fixed duration (e.g., 10 minutes) where windows do not overlap.

- Sliding (Overlapping) Window: Created by adding a sliding interval (e.g., a 10-minute window that slides every 5 minutes). This causes windows to overlap.

- Watermarks: A threshold used to handle late events. A watermark of 10 minutes tells Spark to discard any data arriving more than 10 minutes after the latest event timestamp processed.

Aggregation Code Example:
```python
# Grouping with a 10-minute watermark and 10-minute tumbling window
final_df = words_df \
    .withWatermark("event_time", "10 minutes") \
    .groupBy(
        window(col("event_time"), "10 minutes"), 
        col("word")
    ).count() \
    .select("window.start", "window.end", "word", "count")
```

>--- **Comparing Output Modes and Watermark Effects**



| Feature | Update Mode | Complete Mode |
| :--- | :--- | :--- |
| Behavior | Only outputs rows updated in the current batch. | Rewrites the entire result table to the sink. |
| Late Data | Respects watermarks. Discards data arriving after the cut-off. | Ignores watermarks. Keeps all historical data in memory to print the full table. |
| Memory | Efficient; discards old state based on watermark. | Higher risk of Out of Memory (OOM) errors in stateful processing. |

Practical Example of Late Data:
   Latest Event: 12:17.
   Watermark: 10 minutes (Cut-off time = 12:07).
   Scenario A (Late Data): An event at 11:04 arrives. Update mode discards it, while Complete mode creates a new window for it.
   Scenario B (Acceptable Delay): An event at 12:08 arrives. Because it is after the 12:07 cut-off, both modes update the counts for the 12:00–12:10 window.
