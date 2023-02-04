---
title: "Basics"
date: 2023-02-05T00:11:31+05:45
draft: false
---

#### See all running pods
```bash
docker ps
```

#### See all ran pods
```bash
docker ps -a
```

#### Remove \<none\> (dangling) images
```bash
docker rmi $(docker images -f "dangling=true" -q)
```

#### Remove all containers
```bash
docker rm $(docker ps -aq)
```
#### Remove images conatining keyword
```bash
docker rmi -f $(docker images | grep 'keyword' | awk '{print $1}')
```
