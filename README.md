# PauseAI.info website

[![Netlify Status](https://api.netlify.com/api/v1/badges/628797a4-8d5a-4b5f-94d7-236b4604b23c/deploy-status)](https://app.netlify.com/sites/pauseai/deploys)

SvelteKit website for [PauseAI.info](https://pauseai.info/).

## Creating Articles

You can create and edit most content using [Decap CMS](https://pauseai-cms.netlify.app/), a user-friendly web interface for managing content.

(Some special pages, including the homepage, require editing other Svelte content outside of the CMS.)

### Steps to Create a New Article:

1. Go to [pauseai-cms.netlify.app](https://pauseai-cms.netlify.app/).
2. Log in with a GitHub account.
3. If you are **not authorized to publish content independently**, Decap CMS will prompt you to **fork the repository** before making any changes. Confirm this to create your own copy of the content.
4. Click **"New Post"**.
5. Fill in the fields:
   - **Title**: Enter the title of your post.
   - **Slug**: Define a URL-friendly version of your title.
   - **Description (Optional)**: Provide a brief summary of the post.
   - **Image (Optional)**: Upload an image or insert an image URL.
   - **Author (Optional)**: Add your name if applicable.
   - **Date (Optional)**: Select the publication date, or use "Now" for the current date.
   - **Body**: Enter the main content.
6. Click **"Save"** to store your draft.
7. Update the status as needed:
   - **Draft**: The initial state, for work in progress.
   - **In Review**: Submit the article for review and approval.
   - **Ready**: The article is ready to be published.
8. Decap CMS will automatically create a pull request on GitHub to submit your changes for review.

The article will be published (and localized) automatically after approval.

If you are sufficiently changing prominent text, consider inspecting the relevant automatic localizations/translations as a preview.

## Running locally

```bash
  git clone git@github.com:PauseAI/pauseai-website.git

  # Instead of pnpm you could use npm or yarn

  # Install dependencies
  pnpm install

  # Start development server. If it needs to, it will perform some further development setup before it runs.
  pnpm dev

  # Open http://localhost:37572
```

## Working with locales and other features

To make other development choices, including building locally rather than running under dev, you'll have to set up your development environment.

Start by copying the environment template.

```bash
cp template.env .env
```

The setup script and copied template explain more.

Briefly, some dynamic pages need API keys for service calls, and website translations require more opting in.

We cache content for other locales in a [git repository](https://github.com/PauseAI/paraglide) that you clone locally. That's enough to access existing translations for other locales during local development - set the ones you want in your environment. New/changed translations are generated using LLMs, and you can further opt in to test that locally and even add new locales in `project.inlang/settings.json`, but for day-to-day content changes it is easier to wait for any updated translations to be automatically generated by our pre-production build, and previewed.

## Deployment

The contents of the repository are continuously deployed to Netlify when code is pushed.

You can track the deployment status [here](https://app.netlify.com/sites/pauseai/deploys).

## Collaboration

If you have write access to the repository, please use the **"Squash and merge"** option when merging pull requests. This helps keep the Git history clean and easy to follow.
