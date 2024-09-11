# Data labeling training on cats
<p>At some point while diving deeper into automation processes you are faced with the need for data labeling, although just a couple of weeks ago, the phrases data labeling and you were standing at a party called “Earnings on the Internet” in different rooms. Or it would be better to say that you were standing by the pool, and the data labeling was on the third floor, smoking on the balcony with experts in the field of machine learning. How did we meet? Probably, someone pushed it off the balcony into the pool, and I helped it out, soaking my clothes along the way.</p>

<p>And so, you are sitting in the kitchen, smoking one cigarette for two and trying to figure out what each of you do, and how you could be useful to each other.</p>

<p>In general, it’s not so important why I needed it, but the fact that I succeeded is much more interesting. And now that you are already bored enough (or not), let’s get to the point.</p>

<H2>The task</H2>

<p>The title has the phrase — training on cats — and it is not a metaphor. This is a direct indication of what needs to be done. It is necessary to determine what is depicted in the photo (which animal), and what the customer will do with this information is the third thing.</p>

<p>The task, technically, is as clear as possible. And most importantly, it is as simple as possible. Now we just need to implement it. And we will do this through the data labeling service. Really, I can’t write all sorts of labeling programs manually, right? And I also do not know how to.</p>

<p>In general, we are following the path of simplification.</p>

<p>So, we have the task, we have the solution, let’s get to the details:</p>

<p>At the input, we submit a set of photos and pictures depicting various animals. The task is to get a textual description of the animal depicted in the image as the response. The data volume is big, so uploading this volume manually is not an option. We will send all this via API, for which we write a simple script.</p>

<h2>The script</h2>

<p>First, we need to import the necessary libraries. In our script, we will use requests to make HTTP requests, base64 to encode images, os to work with the file system, and json to process JSON data.</p>

```python
  import requests
import base64
import os
import json
```

<p>A function for creating the task</p>

<p>Now we’ll write a function that will create a task for the server of the data labeling service. It will be the create_task function. It will accept the API URL, project ID, path to the image, and the API key.</p>

<p>Execution Steps:</p>

<ul>
  <li>Opening the image and encoding it in base64.</li>
  <li>Forming the task specification (task_spec) containing the encoded image.</li>
  <li>Creating a payload for the request.</li>
  <li>Setting the request headers, including the API key.</li>
  <li>Sending a POST request to the 2Captcha server.</li>
  <li>Processing the server response and returning the result.</li>
</ul>

```python
  def create_task(api_url, project_id, image_path, api_key):
    try:
        with open(image_path, "rb") as image_file:
            encoded_image = base64.b64encode(image_file.read()).decode('utf-8')
            image_data = f"data:image/jpeg;base64,{encoded_image}"
        
        task_spec = [
            {
                "image_with_animal": image_data
            }
        ]
        
        payload = {
            "project_id": project_id,
            "task_spec": task_spec  # Array with one object inside 
        } 

        headers = { 
            "Content-Type": "application/json",
            "Authorization": f"{api_key}" 
        } 

        print("Data to be sent:", json.dumps(payload, indent=4))  # Logging data before sending 

        response = requests.post(api_url + "/tasks", json=payload, headers=headers)
         
        if response.status_code == 201: 
            print(f"The task was successfully created for the file {image_path}")
            return True 
        else: 
            print(f"Error when creating a task for the file {image_path}: {response.status_code}, {response.text}")
            return False 
    except Exception as e: 
        print(f"An error occurred when creating a task for the file {image_path}: {str(e)}")
        return False
```

<H2>A function to check the validity of the project ID</H2>

<p>During the testing of the script, I had to create a foolproof protection. The validate_project_id function checks whether the specified project ID is correct. It sends a GET request to the service server and returns the verification result.</p>

```python
  def validate_project_id(api_url, project_id, api_key):
    headers = {
        "Authorization": f"{api_key}"
    }
    response = requests.get(f"{api_url}/projects/{project_id}", headers=headers)
    return response.status_code == 200
```

<h2>Image processing function</h2>

<p>Since the images will not be uploaded to the project immediately, but will be fed there in some portions, a function to process and check the images for repetition was required. The process_images function processes all images in the specified directory. It checks the validity of the project ID, reads the images, checks if they have already been sent, and creates tasks for new images.</p>

<h2>Execution Steps:</h2>

<ul>
  <li>Checking the validity of the project ID.</li>
  <li>Loading the history of sent images.</li>
  <li>Iterating through all files in the specified directory.</li>
  <li>Checking whether the file is an image and whether it’s been sent earlier.</li>
  <li>Creating a task for each new image.</li>
  <li>Updating the history of sent images.</li>
</ul>

```python
  def process_images(api_url, project_id, images_dir, api_key):
    try:
        if not validate_project_id(api_url, project_id, api_key):
            print(f"Incorrect `project_id`: {project_id}")
            return 

        sent_images = set() 
        # File for storing sent images 
        history_file = "sent_images.json" 

        # Loading the history of sent images if the file exists 
        if os.path.exists(history_file):
            with open(history_file, "r") as file:
                sent_images = set(json.load(file))

        # Going through all files in the directory 
        for filename in os.listdir(images_dir):
            image_path = os.path.join(images_dir, filename) 
            # Checking if the file is an image and if it's been sent earlier 
            if os.path.isfile(image_path) and filename.lower().endswith(('.png', '.jpg', '.jpeg')) and filename not in sent_images:
                print(f"File processing: {image_path}")
                if create_task(api_url, project_id, image_path, api_key):
                    sent_images.add(filename) 

        # Updating the history of sent images 
        with open(history_file, "w") as file: 
            json.dump(list(sent_images), file) 
            print("Process completed successfully.") 
except Exception as e: 
            print(f"An error occurred while processing images: {str(e)}")
```

<p>The main block of the program</p>

<p>In the main block of the program, we set the parameters: API URL, project ID, directory with images, and the API key. Then we call the process_images function.</p>

```python
  # Example of using the function
if __name__ == "__main__":
    api_url = "http://dataapi.2captcha.com " # Updated API URL for creating tasks 
    project_id = 64 # Replace with your project ID
    images_dir = "C:/images " # Specify the directory with images 
    api_key = "Your API key" # Replace with your API key 

    # Check for images in the directory 
    if not os.path.isdir(images_dir):
        print(f"The directory {images_dir} does not exist")
    else:
        image_files = [f for f in os.listdir(images_dir) if f.lower().endswith(('.png', '.jpg', '.jpeg'))]
        if not image_files:
            print(f"There are no images to process in the directory {images_dir}")
        else: 
            print(f"Found {len(image_files)} images for processing")
            process_images(api_url, project_id, images_dir, api_key)
```

<p>This way I got a Python script that automates the process of creating tasks on the data labeling platform. The script reads the images from the directory, encodes them in base64, sends them to the server, and saves the history of the sent images. You can see the whole script below.</p>

```python
  import requests
import base64
import os
import json

def create_task(api_url, project_id, image_path, api_key):
    try:
        with open(image_path, "rb") as image_file:
            encoded_image = base64.b64encode(image_file.read()).decode('utf-8')
            image_data = f"data:image/jpeg;base64,{encoded_image}"
        
        task_spec = [
            {
                "image_with_animal": image_data
            }
        ]
        
        payload = {
            "project_id": project_id,
            "task_spec": task_spec  # Array with one object inside 
        } 

        headers = { 
            "Content-Type": "application/json",
            "Authorization": f"{api_key}" 
        } 

        print("Data to be sent:", json.dumps(payload, indent=4))  # Logging data before sending 

        response = requests.post(api_url + "/tasks", json=payload, headers=headers)
         
        if response.status_code == 201: 
            print(f"The task was successfully created for the file {image_path}")
            return True 
        else: 
            print(f"An error occurred when creating a task for the file {image_path}: {response.status_code}, {response.text}")
            return False 
    except Exception as e: 
        print(f"An error occurred when creating a task for the file {image_path}: {str(e)}")
        return False

def validate_project_id(api_url, project_id, api_key):
    headers = {
        "Authorization": f"{api_key}"
    }
    response = requests.get(f"{api_url}/projects/{project_id}", headers=headers)
    return response.status_code == 200

def process_images(api_url, project_id, images_dir, api_key):
    try:
        if not validate_project_id(api_url, project_id, api_key):
            print(f"Incorrect `project_id`: {project_id}")
            return 

    sent_images = set() 
    # File for storing sent images 
    history_file = "sent_images.json" 

    # Loading the history of sent images if the file exists 
    if os.path.exists(history_file):
            with open(history_file, "r") as file:
                sent_images = set(json.load(file))

    # Going through all files in the directory 
    for filename in os.listdir(images_dir):
            image_path = os.path.join(images_dir, filename) 
            # Checking if the file is an image and if it's been sent earlier 
            if os.path.isfile(image_path) and filename.lower().endswith(('.png', '.jpg', '.jpeg')) and filename not in sent_images:
                print(f"File processing: {image_path}")
                if create_task(api_url, project_id, image_path, api_key):
                    sent_images.add(filename) 

    # Updating the history of sent images 
    with open(history_file, "w") as file: 
        json.dump(list(sent_images), file) 
    print("Process completed successfully.") 
except Exception as e: 
    print(f"An error occurred while processing images: {str(e)}")

# Example of using the function
if __name__ == "__main__":
    api_url = "http://dataapi.2captcha.com " # Updated API URL for creating tasks 
    project_id = 64 # Replace with your project ID
    images_dir = "C:/images " # Specify the directory with images 
    api_key = "Your API key" # Replace with your API key 

    # Checking for images in the directory 
    if not os.path.isdir(images_dir):
        print(f"The directory {images_dir} does not exist")
    else:
        image_files = [f for f in os.listdir(images_dir) if f.lower().endswith(('.png', '.jpg', '.jpeg'))]
        if not image_files:
            print(f"There are no images to process in the directory {images_dir}")
        else: 
            print(f"Found {len(image_files)} images for processing")
            process_images(api_url, project_id, images_dir, api_key)
```

[<img src="https://img.youtube.com/vi/n4qyOXrnE6A/hqdefault.jpg" width="600" height="300"
/>](https://www.youtube.com/embed/n4qyOXrnE6A)

<h2>Configuring the script</h2>

<p>Now, in order for the script to work, you need to prepare everything:</p>

<p>We create a folder where we create a file with an extension.py and call it what you like. I’m not very imaginative, so it will be script.py</p>

<p>Also, we create a subfolder in the folder where our images will be stored. The path to this folder is written in the script — line 84.</p>

<p>Now we need the API key, API URL, and the project number — lines 85, 82, and 83 in the script, respectively.</p>

<p>We collect all this in the data labeling service.</p>

<p>The API key and the project number are in your dashboard.</p>

![1](https://github.com/user-attachments/assets/15d228ad-080a-4c22-ada1-375791bf7240)

<p>And we take the API URL from the API documentation of the service, but I have already written it for you in the script. However, if you need something more complex than labeling animals, you can study it for your own interest.</p>
<p>Also you need to create the project itself in the service so that there is a place to which you send images. In theory, it is possible to send everything through the API, but even I freaked out and did it all manually. If you want, you can figure out how to send everything through the API on your own.</p>

<p>So, we click on the “Add project” button</p>

![2](https://github.com/user-attachments/assets/94d8c2b0-c309-447e-b15e-10821cba4701)

<p>We fill in the fields “Title”, “Description”, and “Public description”. The difference between a description and a public description is that the first one is a short description, and the public one is a description of the task. Don’t ask me why, it’s beyond my competence.</p>

![3](https://github.com/user-attachments/assets/7ef23b0e-d566-490a-a860-778c64afb84d)

<p>We select the language (1) and create two specifications (2 and 3). (2) — these are the fields for sending images. There are only two options — an image or a text, in our case, we need to send images, so we choose an image.</p>

<p>(3) — These are the fields for the worker, so, in fact, these are the fields where it will write an answer for us (the label of the animal). Since I need it to write me an answer to what kind of animal is depicted in the picture, I use Input. In addition to input, there is also a select, radio, and checkbox. In general, there are plenty to choose from.</p>

<p>In the screenshot below, you can see the “Required” checkbox picked — this is a kind of protection for yourself (double control) — to avoid sending an empty task. That is, if it is checked, the task will not be created until the condition is met (in our case, the presence of an image).</p>

![4](https://github.com/user-attachments/assets/810e968c-23f7-4d4c-a24a-dff34166a257)

<p>There is still the possibility of getting the result of the responses directly to the server, but I did not need it. It will probably be needed soon, but not this time.</p>

![5](https://github.com/user-attachments/assets/2b0ba76d-8e6a-47cf-8aac-b7b1a20af4ff)

<p>Actually, that’s it, we save the project, copy its number and paste it into the script (line 83), and you can run it! We run it with the command <code>python script.py</code> in the developer console.</p>

<p>Then the task flies away to the worker and after it is solved, the answers appear in the personal account in this format</p>

![6](https://github.com/user-attachments/assets/84252bcf-7094-4fc8-9fa9-0f6fec8fed57)

<p>So, this is it, the task is solved.</p>

