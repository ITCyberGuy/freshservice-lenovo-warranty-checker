# freshservice-lenovo-warranty-checker
A Python script that fetches service requests from Freshservice, cross-references asset data, and retrieves warranty information from Lenovo's API.

import requests
from datetime import datetime, timedelta
import json
import csv
import random
import time
import pandas as pd
from tqdm import tqdm
import os  # For environment variables
import sys  # For sys.exit()

# Random sleep function to avoid rate-limiting
def sleep_random_time():
    sleep_time = random.uniform(0, 2)
    time.sleep(sleep_time)

# Get Freshservice API key from environment variable
api_key = os.getenv('FRESHSERVICE_API_KEY')
if not api_key:
    print("Freshservice API key not found. Please set the 'FRESHSERVICE_API_KEY' environment variable.")
    sys.exit(1)

# Input for number of days, with a minimum of 3
while True:
    try:
        num_days = int(input("Enter the number of days (minimum 3): "))
        if num_days < 3:
            print("Number of days must be at least 3.")
        else:
            break
    except ValueError:
        print("Please enter a valid number.")

# Calculate the date `num_days` ago
end_date = datetime.now()
start_date = end_date - timedelta(days=num_days)

# Format the date string in ISO 8601 format
start_date_str = start_date.strftime("%Y-%m-%dT%H:%M:%S")

# Define your API endpoint and authentication details
base_url = "https://yourdomain.freshservice.com/api/v2/tickets"  # Replace 'yourdomain' with your Freshservice domain
username = api_key
password = ""  # Freshservice API uses API key as username; password can be left empty

# Initialize list to store service request IDs
service_request_ids = []

# Fetch tickets in the last `num_days`, using pagination
print("Fetching service request IDs...")
for page in range(1, 11):  # Adjust the range if you expect more pages
    params = {
        "updated_since": start_date_str,
        "per_page": 100,
        "page": page
    }

    response = requests.get(base_url, params=params, auth=(username, password))

    if response.status_code == 200:
        try:
            data = response.json().get('tickets', [])
        except json.decoder.JSONDecodeError:
            print("Unable to parse the JSON response")
            sys.exit(1)

        if not data:
            print(f"No more tickets found on page {page}.")
            break

        print(f"Fetching data from Freshservice page: {page}")
        for ticket in data:
            if isinstance(ticket, dict) and ticket.get("type") == "Service Request":
                service_request_ids.append(ticket["id"])
    else:
        print(f"Error fetching data from page {page}. Status code: {response.status_code}")
        break

    response.close()
    sleep_random_time()

print(f"Total Service Request IDs fetched: {len(service_request_ids)}")

# Lists to store the extracted data
asset_numbers = []
buildings = []
technicians = []

print("Fetching specific replacement data...")

# Fetch detailed data for each service request ID
for idx, ticket_number in enumerate(service_request_ids, start=1):
    print(f"Fetching details for ticket {idx}/{len(service_request_ids)}")
    items_url = f"{base_url}/{ticket_number}/requested_items"

    response = requests.get(items_url, auth=(username, password))
    if response.status_code == 200:
        try:
            requested_items = response.json().get('requested_items', [])
            if requested_items and requested_items[0].get('service_item_id') == 47:
                custom_fields = requested_items[0].get('custom_fields', {})
                asset_numbers.append(custom_fields.get('asset'))
                buildings.append(custom_fields.get('building'))
                technicians.append(custom_fields.get('technician_initials'))
                # Excluded 'full_issue_description' and 'user_s_full_name' to avoid collecting PII
        except json.JSONDecodeError:
            print(f"Unable to parse the JSON response for ticket {ticket_number}")
            continue
        except IndexError:
            print(f"No item data found for ticket {ticket_number}")
            continue
    else:
        print(f"Error fetching details for ticket {ticket_number}. Status code: {response.status_code}")

    response.close()
    sleep_random_time()

# Load asset CSV and cross-reference to get serial numbers
asset_csv_path = r"enter/path/to/your/custom-assets-report.csv"  # Replace with your asset CSV file path

# Load the CSV into a pandas DataFrame
try:
    asset_df = pd.read_csv(asset_csv_path)
except FileNotFoundError:
    print(f"Asset CSV file not found at '{asset_csv_path}'. Please provide the correct path.")
    sys.exit(1)
except Exception as e:
    print(f"An error occurred while reading the asset CSV file: {e}")
    sys.exit(1)

# Normalize asset numbers in the DataFrame for consistent comparison
asset_df['Asset#'] = asset_df['Asset#'].astype(str).str.strip().str.lower()

# Get the serial numbers for assets
serial_numbers = []
for asset in asset_numbers:
    normalized_asset = str(asset).strip().lower()
    row = asset_df.loc[asset_df['Asset#'] == normalized_asset]
    if not row.empty:
        serial_numbers.append(row.iloc[0]['Serial Number'])
    else:
        serial_numbers.append(None)

# Function to get warranty info from Lenovo API
def get_warranty_info(serial_numbers):
    url = 'https://supportapi.lenovo.com/v2.5/warranty'
    token = os.getenv('LENOVO_API_TOKEN')
    if not token:
        print("Lenovo API token not found. Please set the 'LENOVO_API_TOKEN' environment variable.")
        sys.exit(1)

    headers = {
        'ClientID': token,
        'Content-Type': 'application/x-www-form-urlencoded'
    }

    warranties_info = []

    for serial in tqdm(serial_numbers, desc="Fetching warranty information", unit="serial"):
        if serial is None:
            warranties_info.append("No Serial Found")
            continue

        params = {'Serial': serial}
        try:
            response = requests.get(url, headers=headers, params=params)
            if response.status_code == 200:
                data = response.json()
                if 'Warranty' in data:
                    warranty_end = max([warranty.get('End', 'N/A') for warranty in data['Warranty']]) if data['Warranty'] else 'N/A'
                    if warranty_end != 'N/A':
                        warranty_end_date_str = warranty_end.split("T")[0]
                        warranty_end_date = datetime.strptime(warranty_end_date_str, "%Y-%m-%d")

                        months_remaining = (warranty_end_date.year - end_date.year) * 12 + (warranty_end_date.month - end_date.month)
                        warranties_info.append(f"In Warranty ({months_remaining} months remaining)" if months_remaining > 0 else "Out Of Warranty")
                    else:
                        warranties_info.append("Warranty data not found")
                else:
                    warranties_info.append("Warranty data not found")
            else:
                warranties_info.append(f"API Error: {response.status_code}")
        except requests.exceptions.RequestException as e:
            print(f"An error occurred while fetching warranty info for serial '{serial}': {e}")
            warranties_info.append("Error fetching warranty info")
        finally:
            response.close()
        sleep_random_time()

    return warranties_info

# Get warranty information for the serial numbers
warranty_info = get_warranty_info(serial_numbers)

# Write the data to the CSV file
csv_filename = r"enter/path/to/your/output_data.csv"  # Replace with your desired output CSV file path

print("Writing data to CSV...")

try:
    with open(csv_filename, mode='w', newline='', encoding='utf-8') as csv_file:
        writer = csv.writer(csv_file)

        # Write the header row
        writer.writerow(["Asset Number", "Warranty Info", "Building", "Technician"])

        # Write the data rows
        for asset, warranty, build, tech in zip(asset_numbers, warranty_info, buildings, technicians):
            writer.writerow([asset, warranty, build, tech])

except PermissionError:
    print("Permission denied. Please ensure the file is not open or write-protected.")
    sys.exit(1)
except Exception as e:
    print(f"An error occurred while writing to CSV: {e}")
    sys.exit(1)

print(f"Data has been exported to {csv_filename}.")
