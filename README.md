import requests
import json
import warnings
from requests.adapters import HTTPAdapter
from requests.packages.urllib3.util.retry import Retry
import concurrent.futures

# Constants
API_URL = "https://your.tower.server/api/v2"
TOKEN = 'your-token'  # Replace with your OAuth token
JOB_TEMPLATE_ID = 'your_job_template_id'  # Replace with your job template ID
MAX_WORKERS = 30  # Maximum number of workers for large operations

# Suppress only the InsecureRequestWarning
warnings.simplefilter('ignore', requests.packages.urllib3.exceptions.InsecureRequestWarning)

# Setup a requests session with retries and SSL verification disabled
session = requests.Session()
retries = Retry(total=5, backoff_factor=0.1, status_forcelist=[500, 502, 503, 504])
session.mount('http://', HTTPAdapter(max_retries=retries))
session.mount('https://', HTTPAdapter(max_retries=retries))

headers = {
    'Authorization': f'Bearer {TOKEN}',
    'Content-Type': 'application/json'
}

# Function to handle creation of inventory and launching job
def handle_inventory_and_job(server):
    inventory_name = f"load_test_{server}"
    inventory_description = f"Auto-created inventory for server: {server}"
    organization_id = 4  # Adjust as per your organization ID

    # Create Inventory
    payload = {
        "name": inventory_name,
        "description": inventory_description,
        "organization": organization_id
    }
    inventory_response = session.post(f"{API_URL}/inventories/", headers=headers, json=payload, verify=False)
    if inventory_response.status_code in [200, 201]:
        inventory_id = inventory_response.json()['id']
        print(f"Inventory created with ID: {inventory_id} for server: {server}")

        # Launch Job
        job_payload = {
            "inventory": inventory_id,
            "extra_vars": json.dumps({"server": server})  # Example of passing extra vars
        }
        job_response = session.post(f"{API_URL}/job_templates/{JOB_TEMPLATE_ID}/launch/", headers=headers, json=job_payload, verify=False)
        if job_response.status_code in [200, 201, 202]:
            return f"Job launched successfully for server {server} with job ID {job_response.json()['id']}"
        else:
            return f"Failed to launch job for server {server}: {job_response.text}"
    else:
        return f"Failed to create inventory for server {server}: {inventory_response.text}"

# Main execution
if __name__ == "__main__":
    with open("servers.txt", "r") as file:
        servers = [line.strip() for line in file if line.strip()]

    num_workers = min(MAX_WORKERS, len(servers))

    # Using ThreadPoolExecutor to manage parallel execution
    with concurrent.futures.ThreadPoolExecutor(max_workers=num_workers) as executor:
        futures = [executor.submit(handle_inventory_and_job, server) for server in servers]
        for future in concurrent.futures.as_completed(futures):
            print(future.result())
