## Process Design and Future Toplogies

Kubernetes is usually setup with one or more "scheduler" nodes and a number of "worker" nodes. The recommended design of a process for this setup would have an outer loop of futures being distributed to each node in the cluster and then an inner loop of futures performing computation in parallel on the worker nodes. Future topologies are agile to their environment, this means we can run an outer process in parallel on the scheduler then additional processes will run in parallel on the worker. Henrik Bengtsson, the creator of the futures R package has prepared a wonderful overview of this topic, it is recommended that you read it with particular attention to the section on ad-hoc clusters. https://future.futureverse.org/articles/future-3-topologies.html. Below is an example script that would be fully functional and distribute processes efficiently:

```{R}
library(future)
library(dplyr)
library(furrr)

plan(list(
  tweak(cluster, manual = TRUE),
  multisession
))

scheduler_fun <- function(i) {
  df <- data.frame(
    task = i,
    thost = Sys.info()[["nodename"]],
    tpid = Sys.getpid()
  )
  node_fun(df)
}

node_fun <- function(dft) {
  future_map_dfr(1:7, function(x) { # 1:7 run in parallel on worker nodes
    dfb <- data.frame(
      res = mean(rnorm(1*10^x)),
      lhost = Sys.info()[["nodename"]],
      lpid = Sys.getpid(),
      lcores = parallelly::availableCores()
    )
    dplyr::bind_cols(dft, dfb)
  }
  )
}
future_map(1:20, scheduler_fun)

# 1:20, scheduler to workers
# 1:7, workers to worker cores
```

In the case above, the node function would run the future_map_dfr in parallel on the worker node and the schedule node would distribute elements 1:20. 

