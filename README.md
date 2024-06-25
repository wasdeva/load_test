import requests
import concurrent.futures
from datetime import datetime

# Constants
API_URL = "http://your-ansible-tower-url/api/v2"
TOKEN = "your-token"  # Replace this with your actual token
JOB_TEMPLATE_ID = your_job_template_id  # Replace with your job template ID
MAX_WORKERS = 30  # Number of parallel tasks

# Create inventory and launch job
def create_and_launch(server):
    headers = {"Authorization": f"Bearer {TOKEN}"}
    current_time = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")  # Generates a timestamp

    # Create inventory with timestamp and load_test label
    inventory_name = f"load_test_{server}_{current_time}"
    inventory_response = requests.post(
        f"{API_URL}/inventories/",
        json={"name": inventory_name},
        headers=headers
    )
    inventory_id = inventory_response.json()['id']
    
    # Add host to inventory
    requests.post(
        f"{API_URL}/hosts/",
        json={"name": server, "inventory": inventory_id},
        headers=headers
    )
    
    # Launch a job using a job template
    job_response = requests.post(
        f"{API_URL}/job_templates/{JOB_TEMPLATE_ID}/launch/",
        json={"inventory": inventory_id},
        headers=headers
    )
    return f"Launched job for {server}: {job_response.json()}"

# Main script
def main(file_path):
    with open(file_path, 'r') as file:
        servers = [line.strip() for line in file if line.strip()]  # Read each line, strip whitespace
    
    # Use ThreadPoolExecutor to manage parallel execution
    with concurrent.futures.ThreadPoolExecutor(max_workers=MAX_WORKERS) as executor:
        futures = [executor.submit(create_and_launch, server) for server in servers]
        for future in concurrent.futures.as_completed(futures):
            print(future.result())

# Specify the path to your file
file_path = 'servers.txt'  # Path to the file containing server names

if __name__ == "__main__":
    main(file_path)
