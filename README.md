# Eye_seeing_wokflow
GitHub Actions workflow for deploying Astro site to Cloudflare Pages."
name: Pilot - astro pilot soundflare

on:
  # Trigger workflow on pushes to the main branch and manual runs
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read

env:
  PROJECT_PATH: "." # Path to the Astro project; adjust if using subfolders

jobs:
  detect:
    name: Detect Environment
    runs-on: ubuntu-latest
    outputs:
      package_manager: ${{ steps.detect-package-manager.outputs.manager }}
      lockfile: ${{ steps.detect-package-manager.outputs.lockfile }}
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Detect Package Manager
        id: detect-package-manager
        run: |
          if [ -f "${{ github.workspace }}/yarn.lock" ]; then
            echo "manager=yarn" >> $GITHUB_OUTPUT
            echo "lockfile=yarn.lock" >> $GITHUB_OUTPUT
          elif [ -f "${{ github.workspace }}/package.json" ]; then
            echo "manager=npm" >> $GITHUB_OUTPUT
            echo "lockfile=package-lock.json" >> $GITHUB_OUTPUT
          else
            echo "Error: Package manager could not be detected"
            exit 1
          fi

  build:
    name: Build Astro Project
    needs: detect
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: ${{ needs.detect.outputs.package_manager }}
          cache-dependency-path: ${{ env.PROJECT_PATH }}/${{ needs.detect.outputs.lockfile }}

      - name: Install Dependencies
        run: ${{ needs.detect.outputs.package_manager }} install
        working-directory: ${{ env.PROJECT_PATH }}

      - name: Build Project
        run: |
          npx astro build
        working-directory: ${{ env.PROJECT_PATH }}

      - name: Upload Build Artifact
        uses: actions/upload-artifact@v3
        with:
          name: build-output
          path: ${{ env.PROJECT_PATH }}/dist

  deploy:
    name: Deploy to Cloudflare Pages
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download Build Artifact
        uses: actions/download-artifact@v3
        with:
          name: build-output
          path: ./dist

      - name: Deploy to Cloudflare Pages
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          CLOUDFLARE_PROJECT_NAME: ${{ secrets.CLOUDFLARE_PROJECT_NAME }}
        run: |
          curl -X POST \
            "https://api.cloudflare.com/client/v4/accounts/${{ env.CLOUDFLARE_ACCOUNT_ID }}/pages/projects/${{ env.CLOUDFLARE_PROJECT_NAME }}/deployments" \
            -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
            -F "file=@./dist"
