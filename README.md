# MyFirstWebsite

https://docs.aws.amazon.com/AmazonS3/latest/userguide/HostingWebsiteOnS3Setup.html

--------------------------

Here are the steps to host a simple `index.html` website on an Amazon Linux 2023 instance using a Docker container, accessible via an SSH connection:

**Prerequisites:**

  * An Amazon EC2 instance running Amazon Linux 2023.
  * SSH access to the instance.
  * Basic understanding of Linux commands and Docker.

**Steps:**

**1. SSH into your Amazon Linux 2023 instance:**

```bash
ssh -i your-key.pem ec2-user@your-instance-public-ip
```

**2. Update the system and install Docker:**

```bash
sudo yum update -y
sudo yum install docker -y
```

**3. Start the Docker service:**

```bash
sudo systemctl start docker
```

**4. Enable Docker to start on boot (optional but recommended):**

```bash
sudo systemctl enable docker
```

**5. Add your `ec2-user` to the `docker` group (so you don't need `sudo` for Docker commands):**

```bash
sudo usermod -aG docker ec2-user
```

  * **Important:** You'll need to log out and log back in for this group change to take effect. You can simply close your current SSH session and reconnect.

**6. Create your `index.html` file:**

Once you've reconnected via SSH, create a directory for your website and an `index.html` file within it.

```bash
mkdir my-website
cd my-website
nano index.html
```

Paste the following simple HTML content into `index.html` (or your desired content):

```html
<!DOCTYPE html>
<html>
<head>
    <title>My Simple Website</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            margin-top: 50px;
            background-color: #f0f0f0;
            color: #333;
        }
        h1 {
            color: #007bff;
        }
    </style>
</head>
<body>
    <h1>Hello from my Dockerized Website on Amazon Linux 2023!</h1>
    <p>This is a simple test page.</p>
</body>
</html>
```

Save the file and exit `nano` (Ctrl+X, Y, Enter).

**7. Run a Nginx Docker container to serve your website:**

Now, you'll run a Docker container using the Nginx web server, mounting your `my-website` directory as the content source.

```bash
docker run -d -p 80:80 --name my-nginx-website -v ~/my-website:/usr/share/nginx/html nginx
```

Let's break down this command:

  * `docker run`: Command to run a Docker container.
  * `-d`: Runs the container in detached mode (in the background).
  * `-p 80:80`: Maps port 80 on your EC2 instance (host) to port 80 inside the Docker container. This is the standard HTTP port.
  * `--name my-nginx-website`: Assigns a name to your container, making it easier to manage.
  * `-v ~/my-website:/usr/share/nginx/html`: This is the crucial part. It mounts your local `~/my-website` directory (where your `index.html` is) to the `/usr/share/nginx/html` directory inside the Nginx container. Nginx serves content from this directory by default.
  * `nginx`: The name of the Docker image to use (official Nginx image from Docker Hub).

**8. Verify the Docker container is running:**

```bash
docker ps
```

You should see an entry for `my-nginx-website` with status `Up`.

**9. Configure Security Group (AWS Console):**

Even though your container is running, you need to allow inbound HTTP traffic to your EC2 instance from the internet.

1.  Go to the **EC2 Dashboard** in the AWS Management Console.
2.  Navigate to **Instances** and select your Amazon Linux 2023 instance.
3.  In the "Security" tab, click on the **Security Group** associated with your instance.
4.  Click on **Edit inbound rules**.
5.  Click **Add rule**.
6.  For "Type," select **HTTP**.
7.  For "Source," select **Anywhere-IPv4** (0.0.0.0/0) if you want it publicly accessible, or a specific IP range if you want to restrict access.
8.  Click **Save rules**.

**10. Access your website:**

Open your web browser and navigate to your EC2 instance's Public IP address or Public DNS.

`http://your-instance-public-ip/`

You should see your "Hello from my Dockerized Website on Amazon Linux 2023\!" page.

**To stop and remove the container (if needed):**

```bash
docker stop my-nginx-website
docker rm my-nginx-website
```

That's it\! You now have a simple website running locally on your Amazon Linux 2023 instance within a Docker container.
