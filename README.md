# github-actions-frontend

![Image](https://github.com/user-attachments/assets/051a7f5f-2cfc-4c71-98db-01e6e7ee8acf)

## .github/workflows/deploy-prod.yml

```yaml
# .github/workflows/deploy-prod.yml
name: Deploy Production

on:
  push:
    branches:
      - main

env:
  AWS_REGION: 'ap-northeast-2'
  S3_BUCKET: 'github-test-stg.maum.ai'
  DISTRIBUTION_ID: 'E1OH4CAZIA1RV5'
  WEB_DEPLOYMENT_ROLE_ARN: 'arn:aws:iam::139243783662:role/GithubCI-MaumAI-Company'

permissions:
  id-token: write   # This is required for requesting the JWT
  contents: read    # This is required for actions/checkout

jobs:
  build:
    name: Build
    uses: ./.github/workflows/yarn-build.yml
    with:
      node-version: "16.x"

  deploy:
    name: Deploy to Production
    needs: build
    runs-on: ubuntu-latest
    # environment: main  # GitHub Environments에서 main 환경에 등록된 secret 사용
    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: build-artifact
          path: ./build

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ env.WEB_DEPLOYMENT_ROLE_ARN }}
          role-session-name: GitHub_to_AWS_via_FederatedOIDC
          aws-region: ${{ env.AWS_DEFAULT_REGION }}

      - name: Verify AWS Authentication
        run: aws sts get-caller-identity

      - name: deploy to S3
        uses: reggionick/s3-deploy@v4
        with:
          folder: dist
          bucket: ${{ env.S3_BUCKET }}
          bucket-region: ${{ env.AWS_REGION }}
          dist-id: ${{ env.DISTRIBUTION_ID }}
          invalidation: '/'
          delete-removed: true
          private: true
```

## .github/workflows/deploy-stg.yml

```yaml
# .github/workflows/deploy-stg.yml
name: Deploy Staging

on:
  push:
    branches:
      - develop

env:
  AWS_REGION: 'ap-northeast-2'
  S3_BUCKET: 'github-test.maum.ai'
  DISTRIBUTION_ID: 'E1UD2LU1FCY2OK'
  WEB_DEPLOYMENT_ROLE_ARN: 'arn:aws:iam::139243783662:role/GithubCI-MaumAI-Company'

permissions:
  id-token: write   # This is required for requesting the JWT
  contents: read    # This is required for actions/checkout

jobs:
  build:
    name: Build
    uses: ./.github/workflows/yarn-build.yml
    with:
      node-version: "16.x"

  deploy:
    name: Deploy to Staging
    needs: build
    runs-on: ubuntu-latest
    # environment: develop  # GitHub Environments에서 develop 환경에 등록된 secret 사용
    steps:
      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: build-artifact
          path: ./build

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ env.WEB_DEPLOYMENT_ROLE_ARN }}
          role-session-name: GitHub_to_AWS_via_FederatedOIDC
          aws-region: ${{ env.AWS_DEFAULT_REGION }}

      - name: Verify AWS Authentication
        run: aws sts get-caller-identity

      - name: deploy to S3
        uses: reggionick/s3-deploy@v4
        with:
          folder: dist
          bucket: ${{ env.S3_BUCKET }}
          bucket-region: ${{ env.AWS_REGION }}
          dist-id: ${{ env.DISTRIBUTION_ID }}
          invalidation: '/'
          delete-removed: true
          private: true
```

## .github/workflows/yarn-build.yml

```yaml
# .github/workflows/yarn-build.yml
name: Yarn Build

on:
  workflow_call:
    inputs:
      node-version:
        description: "Node.js version to use"
        required: false
        default: "16.x"
        type: string

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [ "${{ inputs.node-version }}" ]

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - name: Install dependencies
        run: yarn install --frozen-lockfile

      - name: Lint code
        run: yarn lint

      - name: Build project
        run: yarn build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: ./build
```