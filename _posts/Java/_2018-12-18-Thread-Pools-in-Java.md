# Executor Framework
1. ThreadPoolExecutor
    - Executor
        - ExecutorService
            - AbstraceExecutorService
                - ThreadPoolExecutor
                    - execute()
                    - submit()
                    - invokeAny()
                    - invokeAll()
                    - shutdown() -- soft shutdown
                    - shutdownNow() -- a waiting ExecutorSErvice will cause the JVM to kee running
1. ScheduledThreadPoolExecutor
    - Executor
        - ExecutorService
            - AbstraceExecutorService
                - ThreadPoolExecutor
                    - ScheduledThreadPoolExecutor
    - Executor
        - ExecutorService
            - ScheduledExecutorService
                - ScheduledThreadPoolExecutor

# Fork/Join Pool Framework
This framework first 'fork' the given task, processing it, and then 'join'.  
- Executor
    - ExecutorService
        - AbstraceExecutorService
            - ForkJoinPool
- Thread
    - ForkJoinWorkerThread
Each of ForkJoinWorkerThread contains double-ended queue (deque), which stores forked tasks.  
By using this deque and Work Stealing Algorithm, Fork/Join Pool framework allows one thread to work for multiple (especially recursive) tasks.  
