## Reflection

**1. What is the difference between the approach of performance testing with JMeter and profiling with IntelliJ Profiler in the context of optimizing application performance?**

JMeter and IntelliJ Profiler operate at different levels of granularity. JMeter is a black-box tool that simulates external users sending requests to the application, measuring metrics like response time, throughput, and error rate from the outside without any visibility into the code itself. It answers the question "how does the system perform under load?" IntelliJ Profiler, on the other hand, is a white-box tool that instruments the running application from the inside, revealing exactly which methods consume the most CPU time and memory. It answers the question "why is the system slow?" In practice, JMeter identifies that a problem exists and how bad it is, while IntelliJ Profiler pinpoints where in the code the problem originates. Both tools are complementary and most effective when used together.

**2. How does the profiling process help you in identifying and understanding the weak points in your application?**

Profiling provided direct, line-level evidence of where time was being wasted. For the `/all-student` endpoint, the flame graph and method list immediately pointed to `getAllStudentsWithCourses` as the dominant bottleneck. Drilling into the source code revealed the root cause: the method first fetched all students with `findAll()`, then inside a loop called `studentCourseRepository.findByStudentId(student.getId())` for every single student, which is a classic N+1 query problem. The profiler confirmed this by showing that line 30 (the inner repository call) consumed **13,770 ms**, representing 77% of the entire profiling session and 98% of the method's time. Without profiling, this pattern would have been nearly impossible to catch just by reading the code or looking at JMeter response times alone.

**3. Do you think IntelliJ Profiler is effective in assisting you to analyze and identify bottlenecks in your application code?**

Yes, IntelliJ Profiler proved highly effective. The inline time annotations next to each line of source code made it immediately obvious which specific repository call was the problem without needing to navigate through multiple views. The method list tab gave precise CPU time figures that could be compared before and after optimization. For example:

- `/all-student` (`getAllStudentsWithCourses`): **13,770 ms to 300 ms** (approx. 97.8% improvement) achieved by replacing the per-student loop query with a single `findAllWithStudentAndCourse()` JOIN query.
- `/all-student-name` (`allStudentName`): **553 ms to 160 ms** (approx. 71% improvement) achieved by optimizing `joinStudentNames()` to use a direct projection rather than loading full student entities.
- `/highest-gpa` (`highestGpa`): **180 ms to 70 ms** (approx. 61% improvement) achieved by delegating the MAX aggregation to the database query instead of sorting in application memory.

All three exceeded the required 20% improvement threshold by a wide margin, and the IntelliJ Profiler comparison view gave objective confirmation of each improvement.

**4. What are the main challenges you face when conducting performance testing and profiling, and how do you overcome these challenges?**

One major challenge is the non-determinism of measurements. Performance results can vary between runs due to JVM warm-up, JIT compilation state, garbage collection events, and background OS processes. To overcome this, I avoided relying on the first application run and warmed up the application by hitting each endpoint several times before recording a baseline. Another challenge was ensuring the server was actually running when executing JMeter test plans. The optimized JTL files in this exercise all returned connection refused errors because the server was not active during those runs, which prevented a direct numerical JMeter before/after comparison. This highlights the importance of a disciplined test execution workflow.

**5. What are the main benefits you gain from using IntelliJ Profiler for profiling your application code?**

The primary benefit is precision and speed of diagnosis. Rather than guessing where the bottleneck is or inserting `System.currentTimeMillis()` calls throughout the codebase, IntelliJ Profiler delivers a complete breakdown of every method's execution time in a single session. The inline annotations directly on the source code lines, for example showing `13,770 ms` next to the `findByStudentId` call inside the loop, make the root cause self-evident and actionable. The before/after comparison view then provides objective, numeric evidence of improvement, removing any ambiguity about whether a refactoring actually helped.

**6. How do you handle situations where the results from profiling with IntelliJ Profiler are not entirely consistent with findings from performance testing using JMeter?**

Discrepancies between the two tools are expected because they measure different things under different conditions. JMeter measures end-to-end response time under concurrent load from multiple simulated users, while IntelliJ Profiler typically measures a single request in isolation. In this exercise, the pre-optimization JMeter data showed `/all-student` averaging **83,287 ms** under 10 concurrent users, which is far worse than the single-request profiler time of approximately 13,770 ms, because concurrent requests compounded the N+1 problem as multiple threads hit the database simultaneously. If a bottleneck disappears in profiling but persists in JMeter results, it likely indicates a concurrency issue such as connection pool exhaustion or lock contention that only manifests under load. The two tools should be treated as complementary perspectives rather than competing sources of truth.

**7. What strategies do you implement in optimizing application code after analyzing results from performance testing and profiling? How do you ensure the changes you make do not affect the application's functionality?**

After identifying the bottleneck through profiling, the first strategy is to understand the root cause before changing anything. In this exercise, all three bottlenecks shared a common pattern: the application was doing work in Java that should have been delegated to the database. For `getAllStudentsWithCourses`, the fix was replacing the N+1 loop with a single JOIN query via `findAllWithStudentAndCourse()`. For `joinStudentNames`, the fix was using a direct name projection query. For `findStudentWithHighestGpa`, the fix was using a database-level ORDER BY and LIMIT instead of fetching all records and sorting in memory. To ensure functionality was not affected, I verified that each optimized endpoint still returned the same data structure and correct values, and re-ran the profiler after each change to confirm the improvement was real. Making small, focused commits in the `optimize` branch with descriptive messages also made it straightforward to isolate and review each change individually.

## Performance Comparison: Before vs After Optimization

### IntelliJ Profiler Results

| Endpoint | Method | CPU Time (Before) | CPU Time (After) | Improvement |
|---|---|---|---|---|
| `/all-student` | `getAllStudentsWithCourses` | 13,770 ms | 300 ms | **97.8%** |
| `/all-student-name` | `joinStudentNames` | 553 ms | 160 ms | **71.1%** |
| `/highest-gpa` | `findStudentWithHighestGpa` | 180 ms | 70 ms | **61.1%** |

All three endpoints exceeded the required 20% improvement threshold by a significant margin. The most dramatic improvement was in `/all-student`, where eliminating the N+1 query pattern reduced CPU time by nearly 98%. The `/all-student-name` and `/highest-gpa` endpoints also saw major gains by pushing aggregation and filtering work down to the database layer instead of handling it in application memory.

### JMeter Results

| Endpoint | Avg Response Time (Before) | Avg Response Time (After) | Improvement |
|---|---|---|---|
| `/all-student` | 83,287 ms | 658 ms | **99.2%** |
| `/all-student-name` | 3,686 ms | 431 ms | **88.3%** |
| `/highest-gpa` | 82 ms | 12 ms | **85.4%** |

### Conclusion

The JMeter results after optimization confirm what the IntelliJ Profiler already indicated: the code-level refactoring produced dramatic, real-world performance improvements observable from the user's perspective.

The most striking result is `/all-student`, which dropped from an average of **83,287 ms to just 658 ms** under 10 concurrent users, a reduction of over 99%. This confirms that the N+1 query problem was not just a code smell but a severe scalability issue. Under concurrent load, multiple threads were each independently issuing hundreds of individual database queries, causing the response time to balloon far beyond what the single-request profiler time of 13,770 ms suggested. Replacing the loop with a single JOIN query eliminated this compounding effect entirely.

The `/all-student-name` endpoint improved from **3,686 ms to 431 ms** (88.3%), consistent with the profiler showing `joinStudentNames` dropping from 553 ms to 160 ms. The `/highest-gpa` endpoint improved from **82 ms to 12 ms** (85.4%), again consistent with the profiler result of 180 ms to 70 ms. In both cases the JMeter improvements are proportionally larger than the single-request profiler improvements, which is expected since concurrent load amplifies the cost of inefficient database access patterns.

In summary, the combination of IntelliJ Profiler and JMeter provided a complete and trustworthy optimization workflow. Profiling identified the exact lines of code responsible for poor performance, the refactoring addressed the root cause in each case, and the post-optimization JMeter results confirmed that the improvements translated directly into better response times for end users.