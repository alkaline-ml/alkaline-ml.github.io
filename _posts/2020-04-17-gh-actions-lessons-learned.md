# GitHub Actions - Lessons Learned

GitHub recently got into the already-crowded CI/CD space with GitHub Actions. For enterprises, you may not want to shake up what already works, but for Open Source Projects, GHA provides a surprising amount of features and an even more enticing price tag: free.

We recently migrated a portion of our builds for pmdarima to GitHub Actions. We're happy with the overall results, but, as early adopters, found some common mistakes that we hope to address here.

## GitHub Actions treats tags as pushes

A common paradigm in software development is to deploy a new version on specific Git tags. With a provider like [CircleCI](https://circleci.com/docs/2.0/workflows/#executing-workflows-for-a-git-tag), you are able to only execute your workflow on a tag event. GitHub Actions treats these as [push](https://help.github.com/en/actions/reference/workflow-syntax-for-github-actions#onpushpull_requestbranchestags) events, because that is technically what they are. This is not a huge problem, but it means you have to change your YAML file from something like this:
  
```
run: deploy.sh
if: github.event_name == 'tag'
```
  
To something like this:
	
```
run: deploy.sh
if: startsWith(github.ref, 'refs/tags')
```

## GitHub Actions will not cancel your run if you push new changes

How many times have you pushed your code to realize you had left a line commented out? Or had some other syntax error? So you immediately push again! CI/CD providers have a number of ways to deal with this:

- [CircleCI gives the option to cancel builds on push](https://circleci.com/docs/2.0/skip-build/#auto-cancelling-a-redundant-build)
- Azure Pipelines has the same behavior _only_ for PRs using the [`autoCancel` feature](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/triggers?view=azure-devops&tabs=yaml#pr-triggers). Otherwise, Azure Pipelines will _not_ cancel you're build, but it will buffer any changes before starting a new build if you use the [`batch` feature](https://docs.microsoft.com/en-us/azure/devops/pipelines/build/triggers?view=azure-devops&tabs=yaml#batching-ci-runs). This means if you push five times while a build is running, it will only trigger a single build later with all five commits

GitHub provides us none of these options. In the above scenario, it will create five new builds for each commit pushed to the source branch. You are able to manually cancel these in the UI, but it will not happen on its own.

It's important to know this, especially if you are building for macOS -- GitHub Actions provides twenty parallel builds on its free tier, however unless you are willing to shell out for Enterprise, [only five of those jobs can be on macOS](https://help.github.com/en/actions/reference/workflow-syntax-for-github-actions#usage-limits). For our specific project, each build triggers four macOS jobs, so we can be very limited if a branch is being pushed to in quick succession.


