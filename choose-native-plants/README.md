# Choose Native Plants Application

This README provides instructions for deploying and managing the Choose Native Plants application in the Kubernetes cluster using Helm.

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
   - Set the tag version to match your release (e.g., "v2.0.3")
   - Set the release title (e.g., "Choose Native Plants v2.0.3")
   - Add release notes describing the changes in this version
   - Click "Publish release"

4. The GitHub Actions workflow will automatically build and push the new Docker image to the GitHub Container Registry with the specified tag.

## Deploying the Application with Helm

Once you've updated the release-values.yaml file and created a GitHub release, you can deploy the application using Helm:

```powershell
# Deploy or upgrade the application using Helm
# Note: This only updates the deployment in the Kubernetes cluster, not your local repository
helm upgrade --install choose-native-plants .\pa-wildflower-selector\helm-chart\ -f .\cfp-sandbox-cluster\choose-native-plants\release-values.yaml -n choose-native-plants
```

**Important**: The `helm upgrade` command only affects your Kubernetes cluster deployment. It doesn't modify any files in your local repository. Make sure you've committed and pushed all changes to GitHub before running this command.

### Helm Release Management

```powershell
# View release history
helm history choose-native-plants -n choose-native-plants

# Rollback to a previous release if needed
helm rollback choose-native-plants [REVISION] -n choose-native-plants

# View release status
helm status choose-native-plants -n choose-native-plants
```

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

## CI/CD Integration with Helm

To integrate Helm into your CI/CD pipeline, update your deployment scripts to use Helm commands instead of direct kubectl commands. Here's a typical workflow:

1. **Update Version**: Update the version in release-values.yaml as part of your CI process
2. **Commit Changes**: Commit the updated release-values.yaml to your repository
3. **Deploy with Helm**: Use the helm upgrade command in your deployment pipeline

```powershell
# Example CI/CD script
$VERSION="2.0.3"  # This would be dynamically set in your CI/CD process

# Update the version in release-values.yaml
(Get-Content .\cfp-sandbox-cluster\choose-native-plants\release-values.yaml) -replace 'tag: "\d+\.\d+\.\d+"', "tag: "$VERSION"" | Set-Content .\cfp-sandbox-cluster\choose-native-plants\release-values.yaml

# Deploy with Helm
helm upgrade --install choose-native-plants .\pa-wildflower-selector\helm-chart\ -f .\cfp-sandbox-cluster\choose-native-plants\release-values.yaml -n choose-native-plants
```

## Accessing the Application

The application is accessible at: https://choose-native-plants.sandbox.k8s.phl.io
