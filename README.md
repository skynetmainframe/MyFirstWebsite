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

------------------

Let's set up a dynamic website on your Amazon Linux 2023 EC2 instance using an Angular frontend and a Spring Boot backend, all containerized with Docker and orchestrated with Docker Compose. This time, the HTML will be served dynamically by Spring Boot.

This setup involves more steps and concepts than the static website, so let's break it down carefully.

**Assumptions:**

  * You have an Amazon Linux 2023 EC2 instance.
  * You have SSH access to the instance.
  * Basic familiarity with Linux commands, Docker, Angular, and Spring Boot.
  * **Crucially, for building Angular and Spring Boot projects, your EC2 instance will need sufficient memory and CPU. A `t2.micro` might struggle; `t2.medium` or `t2.large` is recommended for development/build purposes.**

**High-Level Overview:**

1.  **Prepare EC2 Instance:** Install Docker and Docker Compose.
2.  **Spring Boot Backend:**
      * Set up Java and Maven.
      * Create a simple Spring Boot project.
      * Create a REST endpoint that returns the HTML content.
      * Create a Dockerfile for the Spring Boot application.
3.  **Angular Frontend:**
      * Set up Node.js and Angular CLI.
      * Create a basic Angular project.
      * Create a component that fetches the HTML from the Spring Boot backend.
      * Build the Angular application for production.
      * Create a Dockerfile for the Angular application (using Nginx to serve static files).
4.  **Docker Compose:**
      * Define services for the Angular frontend and Spring Boot backend.
      * Set up networking for inter-container communication.
      * Expose the Angular port to the host.
5.  **Deployment:** Build and run with Docker Compose.
6.  **Security Group:** Open necessary ports in AWS.

-----

**Step 0: SSH into your Amazon Linux 2023 instance**

```bash
ssh -i your-key.pem ec2-user@your-instance-public-ip
```

-----

**Part 1: Prepare EC2 Instance (Install Docker & Docker Compose)**

1.  **Update system and install Docker:**

    ```bash
    sudo yum update -y
    sudo yum install docker -y
    ```

2.  **Start Docker service and enable on boot:**

    ```bash
    sudo systemctl start docker
    sudo systemctl enable docker
    ```

3.  **Add `ec2-user` to `docker` group:**

    ```bash
    sudo usermod -aG docker ec2-user
    ```

      * **IMPORTANT:** Log out and log back in for the group change to take effect:
        ```bash
        exit
        # Then reconnect via SSH
        ssh -i your-key.pem ec2-user@your-instance-public-ip
        ```

4.  **Install Docker Compose:**

    ```bash
    sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
    ```

      * Verify installation:
        ```bash
        docker-compose version
        ```

-----

**Part 2: Spring Boot Backend Setup**

1.  **Install Java and Maven:**

    ```bash
    sudo yum install java-17-amazon-corretto-devel -y # Or your preferred Java version
    sudo yum install maven -y
    ```

      * Verify installations:
        ```bash
        java -version
        mvn -v
        ```

2.  **Create Spring Boot Project Structure:**

    ```bash
    mkdir dynamic-website-app
    cd dynamic-website-app
    mkdir backend frontend
    cd backend
    ```

3.  **Generate Spring Boot Project (Manual Setup or `spring init` if available):**

    Since `spring init` might not be directly available without installing a full SDKMAN or similar, we'll create the basic files manually.

      * **`pom.xml` (Maven Project File):**

        ```xml
        <?xml version="1.0" encoding="UTF-8"?>
        <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
            <modelVersion>4.0.0</modelVersion>
            <parent>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-parent</artifactId>
                <version>3.3.1</version> <relativePath/> </parent>
            <groupId>com.example</groupId>
            <artifactId>dynamic-website-backend</artifactId>
            <version>0.0.1-SNAPSHOT</version>
            <name>dynamic-website-backend</name>
            <description>Dynamic Website Backend with Spring Boot</description>
            <properties>
                <java.version>17</java.version>
            </properties>
            <dependencies>
                <dependency>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-web</artifactId>
                </dependency>
                <dependency>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-devtools</artifactId>
                    <scope>runtime</scope>
                    <optional>true</optional>
                </dependency>
                <dependency>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-test</artifactId>
                    <scope>test</scope>
                </dependency>
            </dependencies>

            <build>
                <plugins>
                    <plugin>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-maven-plugin</artifactId>
                    </plugin>
                </plugins>
            </build>

        </project>
        ```

        Save this as `pom.xml` in the `backend` directory.

      * **Spring Boot Application Class:**
        Create directory structure: `src/main/java/com/example/dynamicwebsitebackend/`

        ```bash
        mkdir -p src/main/java/com/example/dynamicwebsitebackend/
        ```

        Then, create `DynamicWebsiteBackendApplication.java`:

        ```java
        package com.example.dynamicwebsitebackend;

        import org.springframework.boot.SpringApplication;
        import org.springframework.boot.autoconfigure.SpringBootApplication;

        @SpringBootApplication
        public class DynamicWebsiteBackendApplication {

            public static void main(String[] args) {
                SpringApplication.run(DynamicWebsiteBackendApplication.class, args);
            }

        }
        ```

        Save this as `src/main/java/com/example/dynamicwebsitebackend/DynamicWebsiteBackendApplication.java`.

      * **REST Controller:**
        Create `HtmlController.java` in the same directory:

        ```java
        package com.example.dynamicwebsitebackend;

        import org.springframework.web.bind.annotation.GetMapping;
        import org.springframework.web.bind.annotation.RestController;
        import org.springframework.web.bind.annotation.CrossOrigin;

        @RestController
        @CrossOrigin(origins = "*") // Allows requests from any origin for development
        public class HtmlController {

            @GetMapping("/api/dynamic-html")
            public String getDynamicHtml() {
                // This HTML is served dynamically from the backend
                return "<!DOCTYPE html>\n" +
                       "<html>\n" +
                       "<head>\n" +
                       "    <title>My Simple Dynamic Website</title>\n" +
                       "    <style>\n" +
                       "        body {\n" +
                       "            font-family: Arial, sans-serif;\n" +
                       "            text-align: center;\n" +
                       "            margin-top: 50px;\n" +
                       "            background-color: #f0f0f0;\n" +
                       "            color: #333;\n" +
                       "        }\n" +
                       "        h1 {\n" +
                       "            color: #007bff;\n" +
                       "        }\n" +
                       "    </style>\n" +
                       "</head>\n" +
                       "<body>\n" +
                       "    <h1>Hello from my Dockerized Dynamic Website (Spring Boot Backend)!</h1>\n" +
                       "    <p>This content was fetched dynamically.</p>\n" +
                       "</body>\n" +
                       "</html>";
            }
        }
        ```

        Save this as `src/main/java/com/example/dynamicwebsitebackend/HtmlController.java`.

4.  **Build Spring Boot Application:**

    Still in the `backend` directory:

    ```bash
    mvn clean package
    ```

    This will compile your Java code and create a `target/dynamic-website-backend-0.0.1-SNAPSHOT.jar` file.

5.  **Create Dockerfile for Spring Boot:**

    Create `Dockerfile` in the `backend` directory:

    ```dockerfile
    # Use a multi-stage build
    FROM amazoncorretto:17-alpine-jdk AS build

    WORKDIR /app

    COPY pom.xml .
    COPY src ./src

    # Build the application
    RUN mvn clean package -DskipTests

    FROM amazoncorretto:17-alpine-jre

    WORKDIR /app

    # Copy the built JAR from the build stage
    COPY --from=build /app/target/*.jar app.jar

    EXPOSE 8080

    ENTRYPOINT ["java", "-jar", "app.jar"]
    ```

    Save this as `Dockerfile` in the `backend` directory.

-----

**Part 3: Angular Frontend Setup**

1.  **Install Node.js and npm:**

    ```bash
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
    export NVM_DIR="$HOME/.nvm"
    [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
    [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion" # This loads nvm bash_completion

    nvm install 20 # Install Node.js v20 (LTS)
    nvm use 20
    npm install -g @angular/cli
    ```

      * Verify installations:
        ```bash
        node -v
        npm -v
        ng version
        ```

2.  **Create Angular Project:**

    Navigate to the `frontend` directory:

    ```bash
    cd ../frontend
    ng new dynamic-website-frontend --no-standalone --skip-git --routing=false --style=css
    # When prompted, choose 'No' for SSR/SSG
    cd dynamic-website-frontend
    ```

    This will create a new Angular project.

3.  **Update Angular Component to Fetch HTML:**

      * **Modify `src/app/app.component.html`:**

        ```html
        <div [innerHTML]="dynamicHtmlContent"></div>
        ```

        Save this file.

      * **Modify `src/app/app.component.ts`:**

        ```typescript
        import { Component, OnInit } from '@angular/core';
        import { HttpClient } from '@angular/common/http';
        import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

        @Component({
          selector: 'app-root',
          templateUrl: './app.component.html',
          styleUrls: ['./app.component.css']
        })
        export class AppComponent implements OnInit {
          title = 'dynamic-website-frontend';
          dynamicHtmlContent: SafeHtml = '';

          constructor(private http: HttpClient, private sanitizer: DomSanitizer) {}

          ngOnInit(): void {
            this.fetchDynamicHtml();
          }

          fetchDynamicHtml(): void {
            // Note: Use 'backend' as the hostname because of Docker Compose service name
            this.http.get('http://backend:8080/api/dynamic-html', { responseType: 'text' })
              .subscribe(
                (htmlString) => {
                  this.dynamicHtmlContent = this.sanitizer.bypassSecurityTrustHtml(htmlString);
                },
                (error) => {
                  console.error('Error fetching dynamic HTML:', error);
                  this.dynamicHtmlContent = this.sanitizer.bypassSecurityTrustHtml(
                    '<h1>Error loading content. Please check the backend service.</h1>'
                  );
                }
              );
          }
        }
        ```

        Save this file.

      * **Import `HttpClientModule` in `src/app/app.module.ts`:**

        ```typescript
        import { NgModule } from '@angular/core';
        import { BrowserModule, DomSanitizer } from '@angular/platform-browser';
        import { HttpClientModule } from '@angular/common/http'; // Import this

        import { AppComponent } from './app.component';

        @NgModule({
          declarations: [
            AppComponent
          ],
          imports: [
            BrowserModule,
            HttpClientModule // Add this to imports
          ],
          providers: [],
          bootstrap: [AppComponent]
        })
        export class AppModule { }
        ```

        Save this file.

4.  **Build Angular Application for Production:**

    Still in the `dynamic-website-frontend` directory:

    ```bash
    npm install # Install Angular project dependencies
    ng build --configuration=production --output-path=./dist/app-output # Build Angular project
    ```

    This will create a `dist/app-output` directory containing your static Angular files.

5.  **Create Dockerfile for Angular (Nginx):**

    Create `Dockerfile` in the `dynamic-website-frontend` directory (the root of your Angular project):

    ```dockerfile
    # Stage 1: Build the Angular application
    FROM node:20-alpine AS build

    WORKDIR /app

    COPY package.json package-lock.json ./
    RUN npm install --legacy-peer-deps

    COPY . .

    # Build Angular application
    RUN npm run build -- --configuration=production --output-path=./dist/app-output

    # Stage 2: Serve with Nginx
    FROM nginx:alpine

    # Copy the built Angular app from the build stage to Nginx's public directory
    COPY --from=build /app/dist/app-output /usr/share/nginx/html

    # Copy custom Nginx configuration (optional, for SPA routing, etc.)
    # COPY nginx.conf /etc/nginx/conf.d/default.conf

    EXPOSE 80

    CMD ["nginx", "-g", "daemon off;"]
    ```

    Save this as `Dockerfile` in the `frontend/dynamic-website-frontend` directory.

-----

**Part 4: Docker Compose Setup**

1.  **Navigate to the root project directory:**

    ```bash
    cd ../.. # This should bring you to dynamic-website-app
    ```

2.  **Create `docker-compose.yml`:**

    ```yaml
    version: '3.8'

    services:
      backend:
        build:
          context: ./backend
          dockerfile: Dockerfile
        ports:
          - "8080:8080" # Expose Spring Boot port for potential direct access/debugging
        expose:
          - "8080" # Expose internally for frontend to access
        environment:
          SPRING_PROFILES_ACTIVE: prod # Example, if you had different profiles
        networks:
          - app-network

      frontend:
        build:
          context: ./frontend/dynamic-website-frontend # Path to your Angular project root
          dockerfile: Dockerfile
        ports:
          - "80:80" # Map host port 80 to container port 80 (for Nginx)
        depends_on:
          - backend # Ensure backend starts before frontend
        networks:
          - app-network

    networks:
      app-network:
        driver: bridge
    ```

    Save this as `docker-compose.yml` in the `dynamic-website-app` directory.

-----

**Part 5: Build and Run with Docker Compose**

1.  **Build and start the services:**

    Still in the `dynamic-website-app` directory:

    ```bash
    docker-compose up --build -d
    ```

      * `--build`: Forces a rebuild of images (important for initial setup and changes).
      * `-d`: Runs containers in detached mode (in the background).

    This command will:

      * Build the Spring Boot Docker image.
      * Build the Angular (Nginx) Docker image.
      * Create a Docker network.
      * Start the `backend` container.
      * Start the `frontend` container, which will then communicate with the `backend` using the service name `backend`.

2.  **Check container status:**

    ```bash
    docker-compose ps
    ```

    You should see both `backend` and `frontend` services with `Up` status.

3.  **View logs (for debugging):**

    ```bash
    docker-compose logs -f frontend # To see Angular/Nginx logs
    docker-compose logs -f backend  # To see Spring Boot logs
    ```

    Look for any errors, especially related to the Angular app failing to connect to the backend, or the Spring Boot app failing to start.

-----

**Part 6: Configure Security Group (AWS Console)**

1.  Go to the **EC2 Dashboard** in the AWS Management Console.

2.  Navigate to **Instances** and select your Amazon Linux 2023 instance.

3.  In the "Security" tab, click on the **Security Group** associated with your instance.

4.  Click on **Edit inbound rules**.

5.  Click **Add rule**.

6.  For "Type," select **HTTP**.

7.  For "Source," select **Anywhere-IPv4** (0.0.0.0/0) for public access, or a specific IP range.

8.  Click **Save rules**.

      * **Note:** If you expose Spring Boot's 8080 directly for debugging (e.g., `-p "8080:8080"` in `docker-compose.yml`), you would also need to open port 8080 in your security group. However, in this setup, the Angular frontend mediates access, so only port 80 is strictly required for the public.

-----

**Part 7: Access your website**

Open your web browser and navigate to your EC2 instance's Public IP address or Public DNS.

`http://your-instance-public-ip/`

You should see your dynamic HTML content fetched from the Spring Boot backend and rendered by your Angular frontend\!

-----

**Troubleshooting Tips:**

  * **"Error loading content" on frontend:**
      * Check `docker-compose logs -f backend` to ensure Spring Boot started without errors.
      * Check `docker-compose logs -f frontend` for network errors when Angular tries to reach `http://backend:8080/api/dynamic-html`. Ensure the `backend` service is healthy.
      * Verify the `backend` service name in `app.component.ts` matches the `docker-compose.yml` service name exactly.
  * **"Bind: address already in use":** If you get this when `docker-compose up` tries to bind port 80, something else on your EC2 instance is already using it. Follow the troubleshooting steps from my previous answer (check `sudo ss -tuln | grep :80`).
  * **Build failures:** Building Angular and Spring Boot projects on a small EC2 instance can be memory-intensive. If builds fail, consider upgrading your EC2 instance type temporarily (e.g., to `t2.medium` or `t2.large`) during the build process, or build locally and copy the generated `target` and `dist` directories.
  * **CORS Issues:** The `@CrossOrigin(origins = "*")` in the Spring Boot controller is crucial for allowing the Angular frontend to fetch data. In a production environment, you would restrict `origins` to your frontend's actual domain.
  * **Nginx Configuration for Angular (Single Page Application):** For a more complex Angular app with routing, you might need a custom `nginx.conf` in your `frontend/dynamic-website-frontend` directory to handle direct URL access (e.g., sending all non-file requests to `index.html`). For this simple example, it's not strictly necessary.

This detailed guide should get your dynamic Angular + Spring Boot application running on your Amazon Linux 2023 EC2 instance\!
