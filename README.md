# azure_test

## minikube:
# Install/upgrade:
helm upgrade --install test ./helm/my-devops-app \
  -f helm/my-devops-app/values-minikube.yaml

# remove
helm uninstall test
