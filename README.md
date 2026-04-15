# .NET Deployment Guide for EKS

This guide provides a comprehensive approach to deploying a .NET application on Amazon EKS (Elastic Kubernetes Service).

## Prerequisites
- AWS account with permission to create and manage EKS clusters.
- .NET SDK installed.
- Docker installed.
- AWS CLI configured.
- kubectl installed.
- Helm installed.

## Folder Structure
```
/kubernetes-multi-env-app/
│
├── Dockerfile
├── helm/
│   ├── chart-name/
│   │   ├── charts/
│   │   ├── templates/
│   │   ├── values.yaml
│   │   ├── Chart.yaml
│   │   └── .helmignore
│   └── README.md
├── github-actions/
│   ├── ci.yml
│   └── cd.yml
└── k8s-manifests/
    ├── deployment.yaml
    └── service.yaml
```  

## Dockerfile
Create a `Dockerfile` in the root.

```Dockerfile
# Using the official .NET Runtime as a parent image
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
COPY . .
EXPOSE 80

# Build and publish the application
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["YourProjectName.csproj", "./"]
RUN dotnet restore "YourProjectName.csproj"
COPY . .
WORKDIR "/src/.”
RUN dotnet build "YourProjectName.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "YourProjectName.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "YourAssemblyName.dll"]
```

## Helm Chart Setup
Create a directory called `helm/chart-name/` and add a `Chart.yaml`:

```yaml
apiVersion: v2
name: your-chart-name
description: A Helm chart for deploying .NET app
version: 0.1.0
```  

Include your `values.yaml` for configurable parameters:

```yaml
replicaCount: 1
image:
  repository: your-docker-repo/your-image
  tag: latest
service:
  type: ClusterIP
  port: 80
```

## GitHub Actions CI/CD
Create the `.github/workflows/ci.yml`:
```yaml
name: CI
on:
  push:
    branches:
      - main
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Setup .NET
        uses: actions/setup-dotnet@v1
        with:
          dotnet-version: '6.0.x'
      - name: Restore dependencies
        run: dotnet restore
      - name: Build
        run: dotnet build --configuration Release
      - name: Publish
        run: dotnet publish --configuration Release --output ./output
```

Create `cd.yml` for CD:

```yaml
name: CD
on:
  push:
    branches:
      - main
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
      - name: Deploy to EKS
        run: |
          aws eks update-kubeconfig --region your-region --name your-cluster
          helm upgrade --install your-release-name ./helm/chart-name
```

## Kubernetes Deployment Manifests
Create `k8s-manifests/deployment.yaml`:
```yaml
yaml
dapiVersion: apps/v1
kind: Deployment
metadata:
  name: your-app-name
spec:
  replicas: 1
  selector:
    matchLabels:
      app: your-app-name
  template:
    metadata:
      labels:
        app: your-app-name
    spec:
      containers:
        - name: your-container-name
          image: your-docker-repo/your-image:latest
          ports:
            - containerPort: 80
```

And the service definition in `k8s-manifests/service.yaml`:

```yaml
yaml
apiVersion: v1
kind: Service
metadata:
  name: your-service-name
spec:
  type: ClusterIP
  ports:
    - port: 80
  selector:
    app: your-app-name
```  

## Conclusion
Follow these steps to deploy your .NET application to Amazon EKS. Modify parameters as per your application requirements.