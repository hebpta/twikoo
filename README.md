<details>
  <summary>get-latest-release</summary>

```
jobs:
  update_release:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Set up Node.js environment
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Get latest release
        id: get-latest-release
        uses: pozetroninc/github-action-get-latest-release@master
        with:
          repository: twikoojs/twikoo
          excludes: prerelease, draft
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Install jq
        run: sudo apt-get install -y jq

      - name: Update package.json with latest release version
        run: |
          latest_release="${{ steps.get-latest-release.outputs.release }}"
          jq --arg ver "$latest_release" '.dependencies["twikoo-vercel"] = $ver' package.json > package.tmp.json && mv package.tmp.json package.json

      - name: Commit and push changes if necessary
        run: |
          git config user.name "GitHub Action"
          git config user.email "action@github.com"
          git add package.json
          git diff-index --quiet HEAD || git commit -m "v${{ steps.get-latest-release.outputs.release }}"
          git push
```
```
jobs:
  update_master:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout master branch
      uses: actions/checkout@v4
      with:
        ref: master

    - name: Download and unzip main branch
      run: |
        curl -L https://codeload.github.com/walinejs/waline/zip/refs/heads/main -o waline-main.zip
        unzip waline-main.zip -d waline-main
        rm waline-main.zip

    - name: Remove .github folder
      run: rm -rf waline-main/waline-main/.github

    - name: Prepare and commit files
      run: |
        rsync -av waline-main/waline-main/ ./
        rm -rf waline-main
        git config user.name "github-actions"
        git config user.email "github-actions@github.com"
        git add .
        git diff-index --quiet HEAD || git commit -m "Merge branch 'main' of https://github.com/walinejs/waline"
        git push

    - name: Push changes to master branch
      uses: ad-m/github-push-action@master
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        branch: master
```
```
jobs:
  update-version:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
      with:
        ref: master

    - name: Get @waline/vercel version
      id: get_version
      run: |
        echo "Fetching @waline/vercel version from waline repository"
        version=$(curl -s https://raw.githubusercontent.com/walinejs/waline/main/packages/server/package.json | jq -r '.version')
        echo "version=$version" >> $GITHUB_ENV

    - name: Update package.json
      run: |
        echo "Updating @waline/vercel version to ${{ env.version }}"
        jq --arg version "${{ env.version }}" '.dependencies["@waline/vercel"] = $version' package.json > tmp.$$.json && mv tmp.$$.json package.json

    - name: Commit changes
      run: |
        git config user.name "GitHub Action"
        git config user.email "action@github.com"
        git add package.json
        git diff-index --quiet HEAD || git commit -m "v${{ env.version }}"
        git push
```
</details>
<details>
  <summary>Update Release Tags for Multiple Repos</summary>

```
name: Update Release Tags for Multiple Repos

on:
  workflow_dispatch:
  schedule:
    - cron: '30 */6 * * *'

env:
  TARGET_REPO1: 
  SOURCE_REPO1: lobehub/lobe-chat
  TARGET_REPO2: 
  SOURCE_REPO2: ChatGPTNextWeb/ChatGPT-Next-Web

jobs:
  update-tag-lobe-chat:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Target Repository - Lobe Chat
        uses: actions/checkout@v4
        with:
          repository: ${{ env.TARGET_REPO1 }}
          token: ${{ secrets.PAT }}
          path: target-repo-lobe-chat

      - name: Fetch Latest Release from Source Repository - Lobe Chat
        id: get-latest-release-lobe-chat
        uses: pozetroninc/github-action-get-latest-release@master
        with:
          repository: ${{ env.SOURCE_REPO1 }}
          excludes: prerelease, draft
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Configure Git - Lobe Chat
        run: |
          git config --global user.name 'GitHub Action'
          git config --global user.email 'action@github.com'
        working-directory: target-repo-lobe-chat

      - name: Create or Update Tag - Lobe Chat
        run: |
          TAG_NAME="release-${{ steps.get-latest-release-lobe-chat.outputs.release }}"
          git tag -f $TAG_NAME
          git push origin -f $TAG_NAME
        working-directory: target-repo-lobe-chat

      - name: Fetch Release Notes from GitHub API - Lobe Chat
        id: get-release-notes-lobe-chat
        run: |
          TAG_NAME="${{ steps.get-latest-release-lobe-chat.outputs.release }}"
          RELEASE_NOTES=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" https://api.github.com/repos/${{ env.SOURCE_REPO1 }}/releases/tags/$TAG_NAME | jq -r '.body // "No release notes available"')
          echo "RELEASE_NOTES_LobeChat<<EOF" >> $GITHUB_ENV
          echo "$RELEASE_NOTES" >> $GITHUB_ENV
          echo "EOF" >> $GITHUB_ENV
      - name: Create or Update GitHub Release - Lobe Chat
        run: |
          TAG_NAME="release-${{ steps.get-latest-release-lobe-chat.outputs.release }}"
          gh release create "$TAG_NAME" --title "${{ steps.get-latest-release-lobe-chat.outputs.release }}" --notes "${{ env.RELEASE_NOTES_LobeChat }}" --target main || \
          gh release edit "$TAG_NAME" --title "${{ steps.get-latest-release-lobe-chat.outputs.release }}" --notes "${{ env.RELEASE_NOTES_LobeChat }}" --target main
        env:
          GITHUB_TOKEN: ${{ secrets.PAT }}
        working-directory: target-repo-lobe-chat

  update-tag-chatgpt-next-web:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Target Repository - ChatGPT Next Web
        uses: actions/checkout@v4
        with:
          repository: ${{ env.TARGET_REPO2 }}
          token: ${{ secrets.PAT }}
          path: target-repo-chatgpt-next-web

      - name: Fetch Latest Release from Source Repository - ChatGPT Next Web
        id: get-latest-release-chatgpt-next-web
        uses: pozetroninc/github-action-get-latest-release@master
        with:
          repository: ${{ env.SOURCE_REPO2 }}
          excludes: prerelease, draft
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Configure Git - ChatGPT Next Web
        run: |
          git config --global user.name 'GitHub Action'
          git config --global user.email 'action@github.com'
        working-directory: target-repo-chatgpt-next-web

      - name: Extract Release Notes - ChatGPT Next Web
        id: extract-release-notes-chatgpt-next-web
        run: |
          RELEASE_NOTES=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" https://api.github.com/repos/${{ env.SOURCE_REPO2 }}/releases/latest | jq -r '.body // "No release notes available"')
          LAST_LINE=$(echo "$RELEASE_NOTES" | tail -n 1)
          URL=$(echo "$LAST_LINE" | awk '{print $NF}')
          VERSION=$(basename "$URL")
          FORMATTED_LINE="**Full Changelog**: [**$VERSION**]($URL)"
          echo "LAST_LINE=$FORMATTED_LINE" >> $GITHUB_ENV

      - name: Create or Update Tag - ChatGPT Next Web
        run: |
          TAG_NAME="release-${{ steps.get-latest-release-chatgpt-next-web.outputs.release }}"
          git tag -f $TAG_NAME
          git push origin -f $TAG_NAME
        working-directory: target-repo-chatgpt-next-web

      - name: Create or Update GitHub Release - ChatGPT Next Web
        run: |
          TAG_NAME="release-${{ steps.get-latest-release-chatgpt-next-web.outputs.release }}"
          RELEASE_NOTES="${{ env.LAST_LINE }}"
          gh release create "$TAG_NAME" --title "${{ steps.get-latest-release-chatgpt-next-web.outputs.release }}" --notes "$RELEASE_NOTES" --target main || \
          gh release edit "$TAG_NAME" --title "${{ steps.get-latest-release-chatgpt-next-web.outputs.release }}" --notes "$RELEASE_NOTES" --target main
        env:
          GITHUB_TOKEN: ${{ secrets.PAT }}
        working-directory: target-repo-chatgpt-next-web
```
</details>
