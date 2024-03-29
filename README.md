# Manual Coverage Badge ✅
A composite action to manually create a coverage badge for a repository, using [`pytest-cov`](https://pytest-cov.readthedocs.io/en/latest/readme.html) with the [shields.io](https://shields.io/) service.

It will also upload the full html coverage report as an artifact which can be easily deployed to your Github pages.

## Setup
The action has three strict setup requirements:

#### 1. Generate coverage reports using `pytest-cov`
You must have installed [`pytest-cov`](https://pytest-cov.readthedocs.io/en/latest/readme.html) and in your testing step, run the test with the `--cov` and `--cov-report` flags to generate a coverage report.

For example:
```bash
poetry run pytest --cov=<package_name> --cov-report html --cov-report json:<package_name>_coverage.json
```
> **📝 Note:** It's a good idea to specify a unique name per package for the json report (as in the above example), because this filename will be used for the Github gist, and the result might be overwritten if they share the same Github user and filename.

#### 2. Create a Github gist
This action stores the result online as a JSON file in Github gist. Therefore, you must create a Github gist and add the gist id as a secret to your repository.

Go to [https://gist.github.com/](https://gist.github.com/) and create a secret gist. The filename should be the same as the JSON report generated by `pytest-cov` (see above).

Copy the gist id from the url, it should look something like this: `https://gist.github.com/<username>/<gist_id>`.

In your repository on Github, go to `Settings > Secrets and variables > Actions`, create a new repository secret called `GIST_ID` and paste in the gist id.

#### 3. Create a Github personal access token
Go to [https://github.com/settings/tokens](https://github.com/settings/tokens) and generate a personal access token with the `gist` scope. 

Copy the token and save it as another repository secret called `GIST_TOKEN`.

## Usage
See [action.yml](action.yml) for the full documentation for this action's inputs and outputs.

### Generate a simple badge
When the setup is complete, here is how you would use the action to generate a custom coverage badge:

```yaml
jobs:
  test:
    # ...
    steps:
      # ...
      - name: Run tests
        run: poetry run pytest --cov=<package_name> --cov-report json:<package_name>_coverage.json

      - name: Create coverage badge
        uses: simula-consulting/manual-coverage-badge@v1
        with:
          coverage-json: <package_name>_coverage.json
          coverage-gist-token: ${{ secrets.GIST_TOKEN }}
          coverage-gist-id: ${{ secrets.GIST_ID }}
```

The badge will be created and can be found at:
```
https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/<username>/<gist-id>/raw/<your_package>_coverage.json
```

## 🎉 Bonus -  How to deploy the full coverage report to Github pages
If you want to upload a full coverage report to GitHub pages (for example linked to your coverage badge), you can do so relatively easily by adding a few steps.

1. Add  the `--cov-report html` flag when running `pytest`. This will store a folder called `htmlcov` with the full coverage report
2. Upload the `htmlcov` folder as an artifact using [`actions/upload-artifact@v4`](https://github.com/actions/upload-artifact)
3. Deploy the artifact to Github pages using  [`actions/deploy-pages`](https://github.com/actions/deploy-pages) (make sure you have GitHub Pages enabled in your repository settings)

Note that you need to fix permissions to read and upload the coverage report. 

Following the recommended approach from the documentation, this is an example workflow for deploying the coverage report to Github pages:


```yaml
jobs:
  test:
    # ...
    steps:
      # ...
      - name: Run tests
        run: poetry run pytest --cov=<package_name> --cov-report html --cov-report json:<package_name>_coverage.json

      - name: Create coverage badge
        uses: simula-consulting/manual-coverage-badge@v1
        with:
          coverage-json: <package_name>_coverage.json
          coverage-gist-token: ${{ secrets.GIST_TOKEN }}
          coverage-gist-id: ${{ secrets.GIST_ID }}

      - name: Fix permissions for htmlcov
        run: |
          chmod -c -R +rX "htmlcov" | while read line; do
              echo "::warning title=Invalid file permissions automatically fixed::$line"
          done
  
  deploy:
    needs: test

    permissions:
      pages: write
      id-token: write

    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    runs-on: ubuntu-latest

    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

> **⚠️ Warning:** Version `v4` or higher of deploy-pages is required.
