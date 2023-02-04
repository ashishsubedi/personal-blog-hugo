---
title: "Basics"
date: 2023-02-05T00:19:07+05:45
draft: false
---

#### List all authenticated credentials (principals)
```bash
gcloud auth list
```

#### Make another account active
```bash
gcloud config set account `ACCOUNT`
```

#### Activate Service account (if new and no principals)
```bash
gcloud auth activate-service-account --key-file=/path/to/key.json
```

