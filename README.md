# gpt4tun
First to install gpt4free https://github.com/xtekky/gpt4free 
pip install -U g4f

import json
import time
import random
import os
import requests
import gc  # Garbage collector to manage memory
from concurrent.futures import ThreadPoolExecutor, as_completed
from threading import Lock
from datasets import load_from_disk  # Hugging Face library for loading the dataset
from g4f.client import Client  # Importing g4f client

# Load the minipile dataset
def load_minipile_dataset():
    dataset_path = r"" # <==== here  paste path to your local dataset folder example C:\Users\Mega-PC\2\data\minipile
    minipile_data = load_from_disk(dataset_path)  # Load the dataset without streaming
    return minipile_data

dataset = load_minipile_dataset()
dataset_iter = iter(dataset)  # Create an iterator from the dataset

# Ensure the 'saveminipile' folder exists
os.makedirs("saveminipile", exist_ok=True) # here  it will create a foldet called savemiinipile in thei case where requested will be saved as json file 

# Fetch and parse proxies from the given URL
def fetch_proxies(proxy_url):
    response = requests.get(proxy_url)
    return response.text.splitlines()

# Validate proxy by sending a request to the target website
def is_proxy_working(proxy, test_url, verify_ssl=False):  # Disabled SSL verification for optimization
    proxies = {"http": proxy, "https": proxy}
    try:
        response = requests.get(test_url, proxies=proxies, timeout=10, verify=verify_ssl)
        if response.status_code == 200:
            print(f"Proxy {proxy} is working for {test_url}")
            return proxy
    except:
        return None

# Check proxies in parallel but stop once the first working proxy is found
def get_working_proxy(proxies, test_url, max_workers=5, verify_ssl=False):  # Reduced workers to optimize memory
    lock = Lock()
    working_proxy = None

    def check_proxy(proxy):
        nonlocal working_proxy
        if working_proxy is not None:
            return None
        result = is_proxy_working(proxy, test_url, verify_ssl)
        with lock:
            if working_proxy is None and result is not None:
                working_proxy = result
        return result

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        future_to_proxy = {executor.submit(check_proxy, proxy): proxy for proxy in proxies}
        for future in as_completed(future_to_proxy):
            if working_proxy is not None:
                break

    return working_proxy

# Function to translate text using g4f client and save the response
def translate_text_with_g4f(text, row_number, proxy=None, retries=3):
    # Limit the text to the first 500 words
    limited_text = ' '.join(text.split()[:500])

    # Define the translation prompt with explanation on dialect nuances
    prompt_template = (
        "Hey, this is an English text. I want you to translate it into the Tunisian dialect."
        " Please follow these steps:\n"
        "1. Translate step by step and highlight differences between Tunisian dialect and Modern Standard Arabic.\n"
        "2. Highlight sections you are confident translating into the Tunisian dialect.\n"
        "Translate this text into the Tunisian dialect, focusing on capturing the essence of the language nuances:\n\n"
    )
    full_input_text = f"{prompt_template}{limited_text}"

    for attempt in range(retries):
        try:
            # Set the proxy if available
            if proxy:
                os.environ['G4F_PROXY'] = f"http://{proxy}"

            # Create a g4f client and request translation
            client = Client()
            response = client.chat.completions.create(
                model="gpt-4o",
                messages=[{"role": "user", "content": full_input_text}]
            )

            response_text = response.choices[0].message.content
            print(f"Translation for row {row_number} completed.")

            # Print the response for tracking
            print(f"Response for row {row_number}: {response_text}")
            break

        except Exception as e:
            print(f"Error during translation for row {row_number} on attempt {attempt + 1}: {e}")
            response_text = "Error: No response or interaction issue."
            if attempt == retries - 1:
                print(f"Failed to translate row {row_number} after {retries} attempts.")

    # Save the response to a JSON file
    response_data = {
        "prompt": prompt_template,
        "text": limited_text,
        "response": response_text
    }
    
    file_name = f"saveminipile/response_row{row_number}_{random.randint(1000, 9999)}.json"
    with open(file_name, "w", encoding="utf-8") as f:
        json.dump(response_data, f, ensure_ascii=False, indent=4)
    print(f"Response for row {row_number} saved to {file_name}")

# Function to check and restart inactive workers
def check_and_restart_workers(executor, futures, proxies, test_url):
    active_workers = sum(1 for future in futures if not future.done())
    if active_workers < 4:
        print(f"Detected {4 - active_workers} inactive workers. Restarting them...")
        for _ in range(4 - active_workers):
            proxy = get_working_proxy(proxies, test_url)
            if proxy:
                print(f"Switching to proxy: {proxy}")
            else:
                print("No working proxy found. Continuing without proxy.")
                proxy = None

            row = next(dataset_iter)
            row_text = row['text']

            futures.append(executor.submit(translate_text_with_g4f, row_text, random.randint(0, len(dataset) - 1), proxy))

# Main loop to process all rows in the dataset
def main_loop():
    proxies = fetch_proxies("https://raw.githubusercontent.com/officialputuid/KangProxy/KangProxy/https/https.txt")
    test_url = "https://copilot.microsoft.com/"
    proxy = None  # Initialize proxy variable
    lock = Lock()  # Lock for thread-safe operations

    with ThreadPoolExecutor(max_workers=20) as executor:
        futures = []
        request_count = 0
        for i, row in enumerate(dataset):
            if request_count % 10 == 0:
                check_and_restart_workers(executor, futures, proxies, test_url)

            with lock:
                row_text = row['text']

            # Submit the translation task to the executor
            futures.append(executor.submit(translate_text_with_g4f, row_text, i, proxy))
            request_count += 1
        
        # Wait for all futures to complete
        for future in as_completed(futures):
            future.result()

# Start the main loop
main_loop()




#You can chnage number of worker as much your hardawre connection support in my case 20 ThreadPoolExecutor(max_workers=20) as executor:
