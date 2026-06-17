
|                                 |                   |     |
| ------------------------------- | ----------------- | --- |
| **Traditional**                 | **UV Equivalent** |     |
| pip install package             | uv add package    |     |
| python -m venv env              | uv init project   |     |
| pip install -r requirements.txt | uv sync           |     |
cd /home/lab-user
uv init flight-booking-server
cd flight-booking-server
uv add "mcp[cli]"


uv init k8s-error-processor-client