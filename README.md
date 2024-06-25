import requests
import json

# Constants
API_URL = "https://your.tower.server/api/v2"
TOKEN = 'your-token'  # Replace with your OAuth token
JOB_TEMPLATE_ID = 'your_job_template_id'  # Replace with your job template ID

# Headers for authentication
headers = {
    'Authorization': f'Bearer {TOKEN}',
    'Content-Type': 'application/json'
}

# Function to create inventory
def create_inventory(name, description, organization):
    url = f"{API_URL}/inventories/"
    payload = {
        "name": name,
        "description": description,
        "organization": organization
    }
    response = requests.post(url, headers=headers, json=payload)
    if response.status_code in [200, 201]:
        inventory_id = response.json()['id']
        print(f"Inventory created with ID: {inventory_id}")
        return inventory_id
    else:
        raise Exception(f"Failed to create inventory: {response.text}")

# Function to launch a job using a job template
def launch_job_with_inventory(job_template_id, inventory_id, extra_vars=None):
    url = f"{API_URL}/job_templates/{job_template_id}/launch/"
    payload = {
        "inventory": inventory_id,  # Use the newly created inventory ID
        "extra_vars": json.dumps(extra_vars) if extra_vars else {}
    }
    response = requests.post(url, headers=headers, json=payload)
    if response.status_code in [200, 201, 202]:
        return response.json()
    else:
        raise Exception(f"Failed to launch job: {response.text}")

# Main execution
if __name__ == "__main__":
    # Define inventory details
    inventory_name = "New Load Test Inventory"
    inventory_description = "Auto-created inventory for load testing"
    organization_id = 4  # Use the correct organization ID for 'load_test'

    # Create inventory and store the returned inventory ID
    inventory_id = create_inventory(inventory_name, inventory_description, organization_id)

    # Define extra_vars for the job, if any
    extra_vars = {"example_var": "value"}

    # Launch the job using the newly created inventory
    job_response = launch_job_with_inventory(JOB_TEMPLATE_ID, inventory_id, extra_vars=extra_vars)
    print("Job Launched:", job_response)

    # Optionally, extract and print the job ID
    job_id = job_response.get('job', 'No Job ID found')
    print("Job ID:", job_id)
