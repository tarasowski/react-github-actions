## **1. Setup Your EC2 Instance**
Make sure your EC2 instance is:
- Running Ubuntu (or any Linux distribution)
- Has Node.js, Nginx, and PM2 (if using SSR or API) installed
- SSH access is configured using a private key

---

## **2. Create GitHub Actions Workflow**
Inside your React project, create the GitHub Actions workflow file:

📄 `.github/workflows/deploy.yml`
```yaml
name: Deploy React App to EC2

on:
  push:
    branches:
      - main  # Change this to your deployment branch

jobs:
  lint:
    name: Lint the code
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install dependencies
        run: npm ci

      - name: Run Linter
        run: npm run lint

  test:
    name: Run Tests
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install dependencies
        run: npm ci

      - name: Run Tests
        run: npm test -- --watchAll=false

  build:
    name: Build the React App
    runs-on: ubuntu-latest
    needs: test
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install dependencies
        run: npm ci

      - name: Build Project
        run: npm run build

      - name: Upload Build Artifact
        uses: actions/upload-artifact@v4
        with:
          name: build
          path: build/

  deploy:
    name: Deploy to EC2
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Download Build Artifact
        uses: actions/download-artifact@v4
        with:
          name: build
          path: build/

      - name: Deploy to EC2
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          source: "build/*"
          target: "/var/www/react-app"

      - name: Restart Nginx (if needed)
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            sudo systemctl restart nginx
```

---

## **3. Configure GitHub Secrets**
Go to **GitHub Repo → Settings → Secrets and Variables → Actions** and add:

- `EC2_HOST` → Your EC2 public IP
- `EC2_USER` → Typically `ubuntu` or `ec2-user`
- `EC2_SSH_KEY` → Your private SSH key

---

## **4. Configure Nginx on EC2**
On your EC2 instance, set up Nginx to serve the React app:

```bash
sudo nano /etc/nginx/sites-available/react-app
```

Add:
```
server {
    listen 80;
    server_name YOUR_DOMAIN_OR_IP;

    root /var/www/react-app;
    index index.html;
    
    location / {
        try_files $uri /index.html;
    }
}
```

Activate the config:
```bash
sudo ln -s /etc/nginx/sites-available/react-app /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

---

Now, when you push to the **main** branch, the app will be linted, tested, built, and deployed to EC2 automatically! 🚀
