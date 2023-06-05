---
title: "Unveiling the Performance Gap: Vms vs Docker Containers"
date: 2023-06-05T11:04:24+05:30
draft: false
cover: ""
tags:
  - "Docker"
  - "Virtualization"
  - "Containers"
  - "Performance"
categories:
  - "Docker"
  - "Virtualization"
  - "Containers"
  - "Performance"
keywords: ""
description: ""
slug: "unveiling-the-performance-gap-vms-vs-docker-containers"
showFullContent: ""
---

In the world of application deployment, two popular options have emerged: running applications on virtual machines (VMs) and using Docker containers. While both approaches have their merits, performance is a critical factor to consider. In this article, we will compare the performance differences between running an application on a VM and running it in a Docker container.

Understanding Virtual Machines (VMs)
Virtual machines are essentially emulated hardware environments that run on a physical server. Each VM has its own operating system, and multiple VMs can coexist on the same physical server. The VM's resources are allocated from the host machine's pool of resources, such as CPU, memory, and storage. This isolation provides strong security and compatibility with legacy applications.

The Rise of Docker Containers
Docker containers, on the other hand, offer a lightweight alternative to VMs. Containers are isolated environments that run on a shared operating system kernel. Instead of emulating hardware, containers leverage the host machine's operating system, libraries, and resources. Docker containers package an application and its dependencies into a single unit, making them portable and easy to deploy across different environments.

## Benchmarking a FAST API application

```python

import os
from typing import Any, List, Optional, Union

from fastapi import Depends, FastAPI, HTTPException, status
from motor.core import Database
from motor.motor_asyncio import AsyncIOMotorClient
from pydantic import BaseModel, Field

app = FastAPI()


class _Id(BaseModel):
    _oid: str = Field(..., alias="$oid")


class CreatedAt(BaseModel):
    _date: str = Field(..., alias="$date")


class CompaniesSchema(BaseModel):
    _id: _Id
    name: Optional[str]
    permalink: Optional[str]
    crunchbase_url: Optional[str]
    homepage_url: Optional[str]
    blog_url: Optional[str]
    blog_feed_url: Optional[str]
    twitter_username: Optional[str]
    category_code: Optional[str]
    number_of_employees: Optional[int]
    founded_year: Optional[int]
    founded_month: Optional[int]
    founded_day: Optional[int]
    deadpooled_year: Optional[int]
    tag_list: Optional[str]
    alias_list: Optional[str]
    email_address: Optional[str]
    phone_number: Optional[str]
    description: Optional[str]
    created_at: Union[CreatedAt, Any]
    overview: Optional[str]


def get_db():
    db = AsyncIOMotorClient(os.getenv("MONGODB_URI"))
    try:
        yield db.sample_training
    finally:
        db.close()


@app.get("/records", response_model=List[CompaniesSchema])
async def get_records(db: Database = Depends(get_db)):
    try:
        return await db.get_collection("companies").find(limit=100).to_list(length=100)
    except Exception as e:
        print(e)
        return HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Something went wrong",
        )


```

it's a simple application that connects to a MongoDB database and returns a list of 100 records from the companies collection. The application is deployed on a AWS t2.micro instance running Ubuntu 22.04. The instance has 1 vCPU and 1 GB of RAM. The MongoDB database is hosted on MongoDB Atlas. the same application is deployed on a Docker container running on the same instance. The Docker container is built using the following Dockerfile:

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt requirements.txt

RUN pip install -r requirements.txt

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

lets run the application on the VM and see how it performs:

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

using k6 to benchmark the application

```bash
k6 run --vus 10 --duration 30s script.js
```

```javascript
import http from "k6/http";
import { sleep } from "k6";

export default function () {
  http.get("http://<your_ip>:8008/records");
  sleep(1);
}
```

```bash
k6 run --vus 10 --duration 30s script.js

          /\      |‾‾| /‾‾/   /‾‾/
     /\  /  \     |  |/  /   /  /
    /  \/    \    |     (   /   ‾‾\
   /          \   |  |\  \ |  (‾)  |
  / __________ \  |__| \__\ \_____/ .io

  execution: local
     script: script.js
     output: -

  scenarios: (100.00%) 1 scenario, 10 max VUs, 1m0s max duration (incl. graceful stop):
           * default: 10 looping VUs for 30s (gracefulStop: 30s)


     data_received..................: 24 MB 753 kB/s
     data_sent......................: 15 kB 469 B/s
     http_req_blocked...............: avg=1.43ms   min=2.4µs    med=5.5µs    max=24.69ms  p(90)=12.68µs  p(95)=20.29ms
     http_req_connecting............: avg=1.42ms   min=0s       med=0s       max=24.64ms  p(90)=0s       p(95)=20.25ms
     http_req_duration..............: avg=907.8ms  min=435.27ms med=863.01ms max=2.04s    p(90)=1.13s    p(95)=1.27s
       { expected_response:true }...: avg=907.8ms  min=435.27ms med=863.01ms max=2.04s    p(90)=1.13s    p(95)=1.27s
     http_req_failed................: 0.00% ✓ 0        ✗ 162
     http_req_receiving.............: avg=84.02ms  min=46.74ms  med=70.34ms  max=312.12ms p(90)=167.51ms p(95)=183.78ms
     http_req_sending...............: avg=31.83µs  min=8.3µs    med=22.75µs  max=155.2µs  p(90)=60.36µs  p(95)=85µs
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s       p(90)=0s       p(95)=0s
     http_req_waiting...............: avg=823.75ms min=368.48ms med=788.71ms max=1.86s    p(90)=1.03s    p(95)=1.13s
     http_reqs......................: 162   5.096966/s
     iteration_duration.............: avg=1.91s    min=1.43s    med=1.86s    max=3.07s    p(90)=2.13s    p(95)=2.28s
     iterations.....................: 162   5.096966/s
     vus............................: 6     min=6      max=10
     vus_max........................: 10    min=10     max=10


running (0m31.8s), 00/10 VUs, 162 complete and 0 interrupted iterations
default ✓ [======================================] 10 VUs  30s
```

as you can see the application running on the VM is able to handle `5.09 requests per second`.

now lets run the same application on a Docker container and see how it performs:

```bash
docker build -t fastapi .
```

```bash
docker run -d -p 8000:8000 fastapi
```

```bash
k6 run --vus 10 --duration 30s script.js
```

```bash
k6 run --vus 10 --duration 30s script.js

          /\      |‾‾| /‾‾/   /‾‾/
     /\  /  \     |  |/  /   /  /
    /  \/    \    |     (   /   ‾‾\
   /          \   |  |\  \ |  (‾)  |
  / __________ \  |__| \__\ \_____/ .io

  execution: local
     script: script.js
     output: -

  scenarios: (100.00%) 1 scenario, 10 max VUs, 1m0s max duration (incl. graceful stop):
           * default: 10 looping VUs for 30s (gracefulStop: 30s)


     data_received..................: 23 MB 734 kB/s
     data_sent......................: 15 kB 457 B/s
     http_req_blocked...............: avg=1.42ms   min=3µs      med=7.5µs    max=24.98ms  p(90)=16.81µs p(95)=19.87ms
     http_req_connecting............: avg=1.41ms   min=0s       med=0s       max=24.96ms  p(90)=0s      p(95)=19.83ms
     http_req_duration..............: avg=960.38ms min=516.79ms med=925.93ms max=2.03s    p(90)=1.21s   p(95)=1.32s
       { expected_response:true }...: avg=960.38ms min=516.79ms med=925.93ms max=2.03s    p(90)=1.21s   p(95)=1.32s
     http_req_failed................: 0.00% ✓ 0        ✗ 158
     http_req_receiving.............: avg=77.26ms  min=58.23ms  med=68.24ms  max=627.54ms p(90)=74.31ms p(95)=132.6ms
     http_req_sending...............: avg=39.5µs   min=10.5µs   med=27.9µs   max=254.6µs  p(90)=78.65µs p(95)=109.64µs
     http_req_tls_handshaking.......: avg=0s       min=0s       med=0s       max=0s       p(90)=0s      p(95)=0s
     http_req_waiting...............: avg=883.07ms min=418.68ms med=860.07ms max=1.96s    p(90)=1.14s   p(95)=1.23s
     http_reqs......................: 158   4.968316/s
     iteration_duration.............: avg=1.96s    min=1.51s    med=1.92s    max=3.06s    p(90)=2.21s   p(95)=2.33s
     iterations.....................: 158   4.968316/s
     vus............................: 5     min=5      max=10
     vus_max........................: 10    min=10     max=10


running (0m31.8s), 00/10 VUs, 158 complete and 0 interrupted iterations
default ✓ [======================================] 10 VUs  30s

```

the application running on the Docker container is able to handle `4.98 requests per second`.

the performance difference between the two environments is mere `2.5%`.

as you can see, the application running on the VM performed slightly better than the application running on the Docker container. This is because the VM has direct access to the host machine's resources, while the Docker container has to go through the host machine's operating system to access the resources. However, the performance difference is negligible, and the Docker container is still a viable option for deploying applications.

## Why I would choose Docker over VMs

Docker containers are more portable than VMs, which means they can be easily moved from one environment to another. This makes them ideal for deploying applications on cloud platforms, such as AWS or GCP.

Horizontal scaling is easier with Docker containers because they are lightweight and can be easily replicated across multiple servers. This allows you to scale your application horizontally without having to worry about the underlying infrastructure.

Docker containers are more secure than VMs because they are isolated from the host machine's operating system. This means that even if a container is compromised, the host machine will not be affected.

## Conclusion

In this article, we compared the performance differences between running an application on a VM and running it in a Docker container. We found that the application running on the VM performed slightly better than the application running on the Docker container. However, the performance difference was negligible, and the Docker container is still a viable option for deploying applications. We also discussed why I would choose Docker over VMs and why Docker containers are more portable than VMs. I hope you found this article helpful. If you have any questions or comments, please reach out to me. Thank you for reading!
