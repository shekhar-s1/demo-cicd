Demo CI/CD app
==============
Build locally:
  mvn -B clean package
  docker build -t REPLACE_DOCKERUSER/demo-app:local .

Run locally:
  docker run -p 8080:8080 REPLACE_DOCKERUSER/demo-app:local

Kubernetes:
  # Replace image placeholder and apply
  kubectl apply -f k8s/
