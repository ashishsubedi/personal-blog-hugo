---
title: "Debugging"
date: 2023-02-04T20:59:39+05:45
draft: false
---

#### View resource usage (top) of different pods
```bash
kc top pods -n namespace
```

#### Get a shell to the running container
```bash
kc exec --stdin --tty <pod-name> -n namespace -- /bin/bash
```
> Note: The double dash `(--)` separates the arguments you want to pass to the command from the kubectl arguments.

#### List all environment variables in the running container
```bash
kc exec <pod-name> env -n namespace
```
#### Opening a shell when a Pod has more than one container
```bash
kc exec -i -t <pod-name> -n namespace --container main-app -- /bin/bash
```


#### Reference: 
1. https://kubernetes.io/docs/tasks/debug/debug-application/get-shell-running-container

