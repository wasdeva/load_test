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
MAX_WORKERS = 10  # Max concurrent workers

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

# Function to find or create inventory
def find_or_create_inventory(name, description, organization):
    # Try to find the inventory
    response = session.get(f"{API_URL}/inventories/?name={name}", headers=headers, verify=False)
    if response.status_code == 200:
        results = response.json().get('results', [])
        if results:
            return results[0]['id']

    # Create the inventory if not found
    payload = {
        "name": name,
        "description": description,
        "organization": organization
    }
    response = session.post(f"{API_URL}/inventories/", headers=headers, json=payload, verify=False)
    if response.status_code in [200, 201]:
        return response.json()['id']
    else:
        raise Exception(f"Failed to find or create inventory: {response.text}")

# Function to launch a job using a job template
def launch_job_with_inventory(job_template_id, inventory_id, extra_vars=None):
    payload = {
        "inventory": inventory_id,
        "extra_vars": json.dumps(extra_vars) if extra_vars else {}
    }
    response = session.post(f"{API_URL}/job_templates/{job_template_id}/launch/", headers=headers, json=payload, verify=False)
    if response.status_code in [200, 201, 202]:
        return response.json()
    else:
        raise Exception(f"Failed to launch job: {response.text}")

# Main execution
if __name__ == "__main__":
    inventory_name = "Load Test Inventory " + datetime.now().strftime("%Y-%m-%d")  # Using only date
    inventory_description = "Auto-created inventory for load testing"
    organization_id = 4  # Organization ID for 'load_test'

    # Parallel execution of inventory creation and job launch
    with concurrent.futures.ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
        futures = []
        for _ in range(MAX_WORKERS):  # Adjust as needed for your scenario
            futures.append(executor.submit(find_or_create_inventory, inventory_name, inventory_description, organization_id))

        for future in concurrent.futures.as_completed(futures):
            inventory_id = future.result()
            print(f"Using Inventory ID: {inventory_id}")
            job_response = launch_job_with_inventory(JOB_TEMPLATE_ID, inventory_id, extra_vars={"example_var": "value"})
            print("Job Launched:", job_response)
