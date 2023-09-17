# One time
helm repo add devopsnirvanas3 s3://devops-nirvana/helm-charts
helm repo add devops-nirvana https://devops-nirvana.s3.amazonaws.com/helm-charts/

# First, grab the latest published templates
cd helm-chart && ./update-templates-from-foundational-template.sh && cd ..
# NOTE: Run dockerhub build now on master, use the semver hash on an env to test/confirm everything works
# Lint check
helm lint ./helm-chart --set name=test,namespace=test
# Bump version.
export PREVIOUS_RELEASE_TAG=1.0.6 # <---- NOTE: UPDATE ME BEFORE RUNNING THE BELOW
export CI_COMMIT_TAG=1.0.7  # <---- NOTE: UPDATE ME BEFORE RUNNING THE BELOW
sed -i "s/1.0.0/$CI_COMMIT_TAG/g" helm-chart/Chart.yaml
sed -i "s/latest/$CI_COMMIT_TAG/g" helm-chart/values.yaml
# Package
helm package -u ./helm-chart
helm template --namespace infrastructure volume-autoscaler ./helm-chart > volume-autoscaler-${CI_COMMIT_TAG}.yaml

# Publish
export AWS_DEFAULT_REGION=us-east-1
ls -la *.tgz
for PUSH_HELM_CHART_TGZ in $(find ./ -maxdepth 1 -name "*.tgz"); do helm s3 push --acl="public-read" $PUSH_HELM_CHART_TGZ devopsnirvanas3; done
aws s3 cp volume-autoscaler-${CI_COMMIT_TAG}.yaml s3://devops-nirvana/volume-autoscaler/volume-autoscaler-${CI_COMMIT_TAG}.yaml --metadata-directive REPLACE --cache-control max-age=0,no-cache,no-store,must-revalidate --acl public-read --content-type 'text/plain'

# Grab the index.yaml file that is s3:// and make it https://
# rm -f *.tgz # SAVE THIS FILE FOR UPLOADING INTO GITHUB RELEASE!!!
rm -f index.yaml || true
aws s3 cp s3://devops-nirvana/helm-charts/index.yaml ./index.yaml
cat index.yaml | \
gsed "s/s3:/https:/g" | \
gsed "s|devops-nirvana/helm-charts|devops-nirvana.s3.amazonaws.com/helm-charts|g" > indexnew.yaml
diff index.yaml indexnew.yaml
mv -f indexnew.yaml index.yaml
aws s3 cp ./index.yaml s3://devops-nirvana/helm-charts/ --metadata-directive REPLACE --cache-control max-age=0,no-cache,no-store,must-revalidate --acl public-read --content-type 'text/plain'
rm -f index.yaml
# rm -f volume-autoscaler-${CI_COMMIT_TAG}.yaml # SAVE THIS FILE FOR UPLOADING INTO GITHUB RELEASE!!!

# Update version in codebase and README
sed -i "s/$PREVIOUS_RELEASE_TAG/$CI_COMMIT_TAG/g" README.md
# WARNING: Go manually update README to fix up the changelog version area, one will be edited accidentally because of the above SED
git add .github/farley-publish-notes.md README.md
# Reset these two files
git checkout helm-chart/Chart.yaml
git checkout helm-chart/values.yaml
# Review
git status
# Commit
git commit -m "version bump to $CI_COMMIT_TAG"

# Rename files to publish to Github Release
mv volume-autoscaler-${CI_COMMIT_TAG}.tgz volume-autoscaler-helm-chart-${CI_COMMIT_TAG}.tgz
mv volume-autoscaler-${CI_COMMIT_TAG}.yaml volume-autoscaler-kubectl-infrastructure-namespace-${CI_COMMIT_TAG}.yaml

# NOTE: UPLOAD ABOVE FILES TO GITHUB RELEASE
rm -f volume-autoscaler-helm-chart-${CI_COMMIT_TAG}.tgz volume-autoscaler-kubectl-infrastructure-namespace-${CI_COMMIT_TAG}.yaml

# This next one undoes the version setting in the Chart.yamls
git reset --hard HEAD
# Save it upstream
git push
