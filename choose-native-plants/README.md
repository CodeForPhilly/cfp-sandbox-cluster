# Choose Native Plants Application

This README provides instructions for deploying and managing the Choose Native Plants application in the Kubernetes cluster using GitOps.

## Creating a New Release

1. Update the `release-values.yaml` file with the new version number:

```yaml
app:
  image:
    tag: "x.y.z"  # Replace with the new version number
```

2. Commit and push the changes to the repository:

```powershell
git add release-values.yaml
git commit -m "Update Choose Native Plants to version x.y.z"
git push origin main
```

3. Create a new release in GitHub:
   - Go to the [GitHub repository](https://github.com/CodeForPhilly/pa-wildflower-selector)
   - Click on the "Releases" tab
   - Click "Draft a new release"
   - Set the tag version to match your release (e.g., "v2.0.4")
   - Set the release title (e.g., "Choose Native Plants v2.0.4")
   - Add release notes describing the changes in this version
   - Click "Publish release"

4. The GitHub Actions workflow will automatically build and push the new Docker image to the GitHub Container Registry with the specified tag.

## Deploying the Application to Sandbox

The sandbox environment uses a GitOps approach for deployments. When you push changes to the main branch of the cfp-sandbox-cluster repository, the deployment will be automatically triggered.

1. After updating the version in `release-values.yaml` and pushing to main, the GitOps process will detect the change and start a new deployment.

## Managing the Application

### Syncing Images

To sync images with the Linode Object Storage, use this command which automatically finds the current pod:

```powershell
$POD_NAME = kubectl get pods -n choose-native-plants -l app.kubernetes.io/name=choose-native-plants -o jsonpath='{.items[0].metadata.name}'
kubectl exec -it $POD_NAME -n choose-native-plants -c choose-native-plants-app -- /bin/sh /app/scripts/sync-images
```

### Downloading Data (Skip Images)

To download data without images, use this command:

```powershell
$POD_NAME = kubectl get pods -n choose-native-plants -l app.kubernetes.io/name=choose-native-plants -o jsonpath='{.items[0].metadata.name}'
kubectl exec -it $POD_NAME -n choose-native-plants -c choose-native-plants-app -- node download --skip-images
```

### Massaging Data

To process the data, use this command:

```powershell
$POD_NAME = kubectl get pods -n choose-native-plants -l app.kubernetes.io/name=choose-native-plants -o jsonpath='{.items[0].metadata.name}'
kubectl exec -it $POD_NAME -n choose-native-plants -c choose-native-plants-app -- node massage
```

### Viewing Previous Logs

To view the previous logs from the application container:

```powershell
$POD_NAME = kubectl get pods -n choose-native-plants -l app.kubernetes.io/name=choose-native-plants -o jsonpath='{.items[0].metadata.name}'
kubectl logs $POD_NAME -n choose-native-plants -c choose-native-plants-app --previous
```

## Troubleshooting

If you encounter issues with the deployment, check the status of the pods:

```powershell
kubectl get pods -n choose-native-plants
```

For more detailed information about a specific pod:

```powershell
$POD_NAME = kubectl get pods -n choose-native-plants -l app.kubernetes.io/name=choose-native-plants -o jsonpath='{.items[0].metadata.name}'
kubectl describe pod $POD_NAME -n choose-native-plants
```

## CI/CD Integration with GitOps

The sandbox environment uses a GitOps approach for deployments. Here's the typical workflow:

1. **Update Version**: Update the version in release-values.yaml as part of your CI process
2. **Commit Changes**: Commit the updated release-values.yaml to the repository
3. **Push Changes**: Push to the main branch to trigger the GitOps deployment

```powershell
# Example CI/CD script to update version
$VERSION="2.0.4"  # This would be dynamically set in your CI/CD process

# Update the version in release-values.yaml
(Get-Content .\cfp-sandbox-cluster\choose-native-plants\release-values.yaml) -replace 'tag: "\d+\.\d+\.\d+"', "tag: "$VERSION"" | Set-Content .\cfp-sandbox-cluster\choose-native-plants\release-values.yaml

# Commit and push changes
git add .\cfp-sandbox-cluster\choose-native-plants\release-values.yaml
git commit -m "Update Choose Native Plants to version $VERSION"
git push origin main
```

## Accessing the Application

The application is accessible at: https://choose-native-plants.sandbox.k8s.phl.io
