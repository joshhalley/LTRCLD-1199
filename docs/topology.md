# Lab Topology

## Topology Diagram

![Lab Topology](images/topology.jpg)

---

## Device Access (SSH)

Use the following information to access the lab devices over SSH.

> The values below are placeholders – fill them based on your lab environment.

| Device Name | Role           | Management IP | SSH Port | Username | Password |
|------------|----------------|--------------:|---------:|----------|----------|
| c8kv-1     | C8000v Router  |               | 22       |          |          |
| c8kv-2     | C8000v Router  |               | 22       |          |          |
| docker-1   | Docker Host    |               | 22       |          |          |
| jump-1     | Jump Host      |               | 22       |          |          |

Example SSH command:

```code
ssh <username>@<management-ip>
