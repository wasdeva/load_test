import requests
import json
from datetime import datetime
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

# Function to find or create inventory and add a host to it
def find_or_create_inventory_and_add_host(server):
    inventory_name = f"load_test_{server}"
    url = f"{API_URL}/inventories/?name={inventory_name}"
    response = session.get(url, headers=headers, verify=False)
    results = response.json().get('results', [])

    if results:
        inventory_id = results[0]['id']
        print(f"Using existing inventory ID {inventory_id} for server: {server}")
    else:
        # If inventory does not exist, create it
        payload = {
            "name": inventory_name,
            "description": f"Auto-created inventory for server: {server}",
            "organization": 4  # Adjust as per your organization ID
        }
        response = session.post(f"{API_URL}/inventories/", headers=headers, json=payload, verify=False)
        if response.status_code in [200, 201]:
            inventory_id = response.json()['id']
            print(f"Inventory created with ID: {inventory_id} for server: {server}")
        else:
            raise Exception(f"Failed to create inventory for server {server}: {response.text}")

    # Add the server as a host to the inventory
    host_payload = {
        "name": server,
        "inventory": inventory_id,
        "enabled": True
    }
    host_response = session.post(f"{API_URL}/hosts/", headers=headers, json=host_payload, verify=False)
    if host_response.status_code not in [200, 201]:
        raise Exception(f"Failed to add host {server} to inventory ID {inventory_id}: {host_response.text}")
    
    return inventory_id

# Function to launch a job using a job template
def launch_job(inventory_id, server):
    job_payload = {
        "inventory": inventory_id,
        "extra_vars": json.dumps({"server": server})  # Example of passing extra vars
    }
    response = session.post(f"{API_URL}/job_templates/{JOB_TEMPLATE_ID}/launch/", headers=headers, json=job_payload, verify=False)
    if response.status_code in [200, 201, 202]:
        return f"Job launched successfully for server {server} with job ID {response.json()['id']}"
    else:
        return f"Failed to launch job for server {server}: {response.text}"

# Main execution
if __name__ == "__main__":
    with open("servers.txt", "r") as file:
        servers = [line.strip() for line in file if line.strip()]

    num_workers = min(MAX_WORKERS, len(servers))

    # Using ThreadPoolExecutor to manage parallel execution
    with concurrent.futures.ThreadPoolExecutor(max_workers=num_workers) as executor:
        futures = []
        for server in servers:
            future = executor.submit(find_or_create_inventory_and_add_host, server)
            futures.append((future, server))

        for future, server in futures:
            inventory_id = future.result()
            job_result = launch_job(inventory_id, server)
            print(job_result)
