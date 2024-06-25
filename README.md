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
SERVERS_PER_INVENTORY = 5  # Number of servers per inventory, default is 1

# Suppress only the InsecureRequestWarning
warnings.simplefilter('ignore', requests.packages.urllib3.exceptions.InsecureRequestWarning)

# Setup a requests session with retries and SSL verification disabled
session = requests.Session()
retries = Retry(total=5, backoff_factor=1, status_forcelist=[429, 500, 502, 503, 504])
session.mount('http://', HTTPAdapter(max_retries=retries))
session.mount('https://', HTTPAdapter(max_retries=retries))

headers = {
    'Authorization': f'Bearer {TOKEN}',
    'Content-Type': 'application/json'
}

# Function to create inventory and add hosts
def create_inventory_with_hosts(servers, inventory_index):
    inventory_name = f"load_test_group_{inventory_index}"
    description = "Auto-created inventory for batch server management"
    payload = {
        "name": inventory_name,
        "description": description,
        "organization": 4  # Adjust as per your organization ID
    }
    response = session.post(f"{API_URL}/inventories/", headers=headers, json=payload, verify=False)
    if response.status_code in [200, 201]:
        inventory_id = response.json()['id']
        print(f"Inventory created with ID: {inventory_id} for servers: {', '.join(servers)}")
        for server in servers:
            add_host_to_inventory(server, inventory_id)
        return inventory_id
    else:
        raise Exception(f"Failed to create inventory for servers: {', '.join(servers)}: {response.text}")

# Function to add a host to an inventory
def add_host_to_inventory(server, inventory_id):
    host_payload = {
        "name": server,
        "inventory": inventory_id,
        "enabled": True
    }
    host_response = session.post(f"{API_URL}/hosts/", headers=headers, json=host_payload, verify=False)
    if host_response.status_code not in [200, 201]:
        print(f"Failed to add host {server} to inventory ID {inventory_id}: {host_response.text}")

# Function to launch a job using a job template
def launch_job(inventory_id, server_group):
    job_payload = {
        "inventory": inventory_id,
        "extra_vars": json.dumps({"servers": server_group})  # Passing list of servers as extra vars
    }
    response = session.post(f"{API_URL}/job_templates/{JOB_TEMPLATE_ID}/launch/", headers=headers, json=job_payload, verify=False)
    if response.status_code in [200, 201, 202]:
        return f"Job launched successfully for servers {', '.join(server_group)} with job ID {response.json()['id']}"
    else:
        return f"Failed to launch job for servers {', '.join(server_group)}: {response.text}"

# Main execution
if __name__ == "__main__":
    with open("servers.txt", "r") as file:
        servers = [line.strip() for line in file if line.strip()]

    # Splitting servers into groups
    server_groups = [servers[i:i + SERVERS_PER_INVENTORY] for i in range(0, len(servers), SERVERS_PER_INVENTORY)]
    inventory_index = 0

    # Processing server groups
    with concurrent.futures.ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
        futures = []
        for server_group in server_groups:
            inventory_index += 1
            future = executor.submit(create_inventory_with_hosts, server_group, inventory_index)
            futures.append((future, server_group))

        for future, server_group in futures:
            inventory_id = future.result()
            job_result = launch_job(inventory_id, server_group)
            print(job_result)
