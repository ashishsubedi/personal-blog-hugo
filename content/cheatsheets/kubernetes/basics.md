---
title: "Basics"
date: 2023-02-04T23:32:43+05:45
draft: false 
---
#### Alias kubectl to kc for easy typing

```bash
alias kc="kubectl"
```
#### List all the available namespaces
```bash
kc get namespace
```

#### Get all pods running in namespace
```bash
kc get pods -n namespace
```

#### Get all pods running in namespace
```bash
kc logs <pod> -n namespace
```

#### Get all resource running in namespace
```bash
kc logs resource/<resource_name> -n namespace
Eg: kc logs services/xyz -n namespace
```

#### Alias: List all the pods and see the logs of selected (requires fzf)
```bash
alias kcl='kubectl get pods -n <namespace> | fzf | awk '\''{print $1}'\'' | xargs kubectl logs -n <namespace>'
```

#### Alias: List all contexts and switch between them (requires fzf)

```bash
alias kx='kubectl config get-contexts | awk '\''{print $2}'\'' | fzf | xargs kubectl config use-context'
```

